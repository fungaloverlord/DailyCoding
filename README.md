# build_graph.rb
# Standalone graph generator for Jekyll sites with docs in _docs/**.
# Produces assets/graph.json with nodes + edges for a graph view.
#
# Supported link types
# ─────────────────────────────────────────────────────────────────────────────
# Inline wikilinks   (body)   [[target]]  [[target|Label]]  [[target#heading]]
# Inline MD links    (body)   [Label](path/to/doc.md)
# Reference usages   (body)   [Label][ref-id]  [Label][]  [label] (shortcut)
# Reference defs     (footer) [ref-id]: path/to/doc.md "Optional title"
# ─────────────────────────────────────────────────────────────────────────────

require "json"
require "pathname"
require "set"
require "yaml"

DOCS_DIR          = "_docs"
OUT_FILE          = File.join("assets", "graph.json")
INCLUDE_MISSING   = false   # true → ghost nodes for unresolvable links
EXTS              = %w[.md .markdown .mdown .mkdn .mkd].freeze

# ── Front-matter helpers ──────────────────────────────────────────────────────

def parse_front_matter(text)
  return {} unless text.start_with?("---\n", "---\r\n")

  yaml_lines = []
  text.lines[1..].each do |line|
    break if line.strip == "---"
    yaml_lines << line
  end
  YAML.safe_load(yaml_lines.join,
                 permitted_classes: [],
                 permitted_symbols: [],
                 aliases: true) || {}
rescue
  {}
end

def strip_front_matter(text)
  return text unless text.start_with?("---\n", "---\r\n")

  lines = text.lines
  return text if lines.size < 3

  end_idx = nil
  lines[1..].each_with_index do |line, i|
    if line.strip == "---"
      end_idx = i + 1
      break
    end
  end

  end_idx ? lines[(end_idx + 1)..].join : text
end

# ── Link extractors ───────────────────────────────────────────────────────────

# [[target]]  [[target|Label]]  [[target#heading]]
# Ignores embeds ![[…]]
def extract_wikilinks(body)
  body.scan(/(?<!!)\[\[([^\]]+)\]\]/).map do |m|
    raw    = m[0].strip
    target, label = raw.split("|", 2).map { |s| s&.strip }
    { target: target, label: label, source: :wikilink }
  end
end

# Inline markdown links: [Label](href)
# Skips plain URLs (http/https/mailto) — those are never internal docs.
def extract_inline_links(body)
  body.scan(/\[([^\]]*)\]\(([^)]+)\)/).filter_map do |label, href|
    href = href.strip.split(/\s+/, 2)[0]   # drop optional "title" part
    next if href.match?(/\Ahttps?:|mailto:/i)
    { target: href, label: label.strip, source: :inline }
  end
end

# Reference-link definitions (typically footer lines):
#   [ref-id]: path/to/file.md
#   [ref-id]: path/to/file.md "Title"
#   [ref-id]: path/to/file.md 'Title'
#   [ref-id]: path/to/file.md (Title)
# Returns Hash { normalised_label => href }
REF_DEF_RE = /
  ^[ ]{0,3}               # optional indent (≤3 spaces per CommonMark)
  \[([^\]]+)\]:           # [label]:
  \s+                     # whitespace
  (\S+)                   # href (no spaces)
  (?:                     # optional title
    \s+
    (?:"[^"]*"|'[^']*'|\([^)]*\))
  )?
/x

def extract_ref_definitions(text)
  defs = {}
  text.scan(REF_DEF_RE) do |label, href|
    next if href.match?(/\Ahttps?:|mailto:/i)
    defs[label.strip.downcase] = href.strip
  end
  defs
end

# Reference-link usages in the body, resolved against ref_defs:
#   [Label][ref-id]   explicit ref
#   [Label][]         collapsed ref  (label IS the ref key)
#   [label]           shortcut ref   (label IS the ref key; only if it matches a def)
# We purposely ignore footnote-style [^...] markers.
def extract_ref_links(body, ref_defs)
  return [] if ref_defs.empty?

  links = []

  # [text][ref-id] and [text][]
  body.scan(/\[([^\]^][^\]]*)\]\[([^\]]*)\]/) do |text_part, ref_id|
    key  = (ref_id.strip.empty? ? text_part : ref_id).strip.downcase
    href = ref_defs[key]
    links << { target: href, label: text_part.strip, source: :ref_link } if href
  end

  # Shortcut [label] — only when the label matches a known definition.
  # Use a negative look-ahead/behind to avoid matching [text](…) and [text][…].
  body.scan(/\[([^\]^][^\]]*)\](?!\[|\()/) do |m|
    key  = m[0].strip.downcase
    href = ref_defs[key]
    links << { target: href, label: m[0].strip, source: :shortcut } if href
  end

  links
end

# Combine all link types for a single document.
# Returns an array of { target: String, label: String|nil, source: Symbol }
def all_links(raw_text)
  fm_stripped = strip_front_matter(raw_text)
  ref_defs    = extract_ref_definitions(raw_text)   # scan full text incl. footer

  extract_wikilinks(fm_stripped) +
    extract_inline_links(fm_stripped) +
    extract_ref_links(fm_stripped, ref_defs)
end

# ── Path normalisation ────────────────────────────────────────────────────────

# Strips anchor, trailing slash, and normalises separators.
def base_target(raw)
  return nil if raw.nil? || raw.empty?

  t = raw.tr("\\", "/")       # Windows paths
  t = t.split("#", 2)[0]      # remove #anchor
  t = t.sub(/\.(md|markdown|mdown|mkdn|mkd)\z/i, "")   # strip extension
  t = t.gsub(/\/+$/, "")      # trailing slash
  t.empty? ? nil : t
end

# Resolve a (possibly relative) link target to a docs-root-relative ID.
# source_id  – ID of the linking document (e.g. "guides/intro")
# target_raw – normalised target string (already base_target-processed)
def resolve_id(source_id, target_raw, path_for_id, basename_to_ids)
  return nil if target_raw.nil?

  # 1) Absolute-style: no leading dot → try direct ID lookup first
  unless target_raw.start_with?(".")
    return target_raw if path_for_id.key?(target_raw)
  end

  # 2) Relative path: resolve against the source document's directory
  source_dir = source_id.include?("/") ? File.dirname(source_id) : "."
  candidate  = Pathname.new(source_dir).join(target_raw).cleanpath.to_s
  candidate  = candidate.tr("\\", "/").sub(%r{\A\./}, "")
  return candidate if path_for_id.key?(candidate)

  # 3) Basename-only fuzzy match (unique basename → safe to resolve)
  basename   = File.basename(target_raw)
  candidates = basename_to_ids[basename]
  return candidates[0] if candidates.length == 1

  nil   # unresolvable
end

# ── Main ──────────────────────────────────────────────────────────────────────

docs_root = Pathname.new(DOCS_DIR)
unless docs_root.directory?
  warn "ERROR: #{DOCS_DIR} folder not found."
  exit 1
end

# 1) Discover docs
docs = docs_root.glob("**/*").select { |p| p.file? && EXTS.include?(p.extname.downcase) }

# 2) Build ID maps
id_for_path      = {}
path_for_id      = {}
basename_to_ids  = Hash.new { |h, k| h[k] = [] }

docs.each do |path|
  rel = path.relative_path_from(docs_root).to_s.tr("\\", "/")
  id  = rel.sub(/#{Regexp.escape(path.extname)}$/i, "")

  id_for_path[path.to_s] = id
  path_for_id[id]        = path.to_s
  basename_to_ids[File.basename(id)] << id
end

# 3) Build nodes + edges
nodes     = []
edges_set = Set.new
link_log  = Hash.new { |h, k| h[k] = { total: 0, resolved: 0, by_type: Hash.new(0) } }

docs.each do |path|
  id  = id_for_path[path.to_s]
  raw = path.read(encoding: "utf-8")

  fm    = parse_front_matter(raw)
  title = fm["title"] || fm["name"] || File.basename(id)
  url   = "/docs/#{id}.md"

  nodes << { id: id, label: title.to_s, path: url }

  links = all_links(raw)
  link_log[id][:total] = links.size

  links.each do |link|
    target_base = base_target(link[:target])
    next if target_base.nil?

    link_log[id][:by_type][link[:source]] += 1

    resolved = resolve_id(id, target_base, path_for_id, basename_to_ids)

    if resolved
      edges_set << [id, resolved].freeze
      link_log[id][:resolved] += 1
    elsif INCLUDE_MISSING
      edges_set << [id, target_base].freeze
    end
  end
end

# 4) Ghost nodes for unresolvable links (opt-in)
if INCLUDE_MISSING
  existing_ids = nodes.map { |n| n[:id] }.to_set
  edges_set.each do |_from, to|
    next if existing_ids.include?(to)
    nodes << { id: to, label: File.basename(to), path: "/docs/#{to}/", missing: true }
    existing_ids.add(to)
  end
end

edges = edges_set.map { |from, to| { from: from, to: to } }
graph = { nodes: nodes, edges: edges, generated_at: Time.now.utc.iso8601 }

# 5) Write output
out_dir = File.dirname(OUT_FILE)
Dir.mkdir(out_dir) unless Dir.exist?(out_dir)
File.write(OUT_FILE, JSON.pretty_generate(graph))

# 6) Summary
puts "Wrote #{OUT_FILE} (#{nodes.size} nodes, #{edges.size} edges)\n\n"
puts "%-40s %6s %8s  %s" % %w[Document Links Resolved By-type]
puts "-" * 72
link_log.sort.each do |id, data|
  by_type = data[:by_type].map { |t, n| "#{t}=#{n}" }.join(", ")
  puts "%-40s %6d %8d  %s" % [id, data[:total], data[:resolved], by_type]
end
