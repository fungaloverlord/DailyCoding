def get_htsno_data(code):
    api = f"https://hts.usitc.gov/reststop/getRates?htsno={code}&keyword={code}"
    response = requests.get(api)
    data = response.json()
    global pbar
    for i in data:
        if i.get('htsno') == code:
            if i.get('footnotes') == []:
                for i in data:
                    if i.get('htsno') == code[:10]:
                        footnotes = i.get('footnotes')
                        foot = []
                        for footnote in footnotes:
                            if "general" in footnote.get('columns'):
                                foot.append(footnote.get('value'))
                        pbar.update(1)
                        return foot
            else:
                for i in data:
                    if i.get('htsno') == code:
                        footnotes = i.get('footnotes')
                        foot = []
                        for footnote in footnotes:
                            if "general" in footnote.get('columns'):
                                value = footnote.get('value').strip(' ')
                                foot.append(value)
                        pbar.update(1)
                        return foot
#get_codes(url,csv_file)
