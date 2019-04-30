# 引用需要用到的模块
import requests
import re
import bs4
import matplotlib as plt
import importlib
import pdb
import os
import sys
import csv
import time
import json
import winsound
sys.getdefaultencoding()  #默认编码是“utf-8”

"""这是一个用于爬取学科网各年级各学科各种材料信息的定向爬虫
旨在为一线的老师能够根据数据关于下载量，阅读量及价格的分析对比选出最优材料
"""
class SpiderXueKeWang(object):
    
    def __init__(self):
        self.headers = {
        # 把cookies填进去
            "Cookie":cookies,
            'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 \
            (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36'}
        self.courses_lst_primary = {"语文":"yw","数学":"sx","英语":"yy","科学":"kx","美术":"ms","音乐":"yinyue",
                                    "品德":"pd","信息技术":"xx","体育":"ty"}
        self.courses_lst_middle = {"语文":"yw","数学":"sx","英语":"yy","物理":"wl","化学":"hx","生物":"sw","政治":"zz",
                                   "历史":"ls","地理":"dl","综合":"zh","科学":"kx"}
        self.courses_lst_high = {"语文":"yw","数学":"sx","英语":"yy","物理":"wl","化学":"hx","生物":"sw","政治":"zz",
                                   "历史":"ls","地理":"dl","综合":"zh","通用":"qt"}
        self.sources_kinds_primary = {"教案":"type-2","试题":"type-3","同步扩展":"type-4",
                                 "素材资料":"type-5","视频":"type-30"}
        self.sources_kinds_middle_high = {"试题试卷":"type-1","教案":"type-2","课件":"type-3","素材":"type-4",
                                     "视频":"type-6","备课综合":"type-7","学案/导学案":"type-8","作业":"type-9"}
        self.grades_dict = {"小学":[self.courses_lst_primary,self.sources_kinds_primary],
                           "初中":[self.courses_lst_middle,self.sources_kinds_middle_high],
                           "高中":[self.courses_lst_high,self.sources_kinds_middle_high]
                          }
        self.s = requests.Session()
        self.__API = """http://api.xdaili.cn/xdaili-api//greatRecharge/getGreatIp?spiderId=3c44943c2dfa4a80b709090f6e4f7f9f&orderno=YZ2019313213eeDYdo&returnType=2&count=4"""
        self.time = 1
        self.duration = 1000  # millisecond
        self.freq = 440  # Hz
# 构造爬取目标网址
    def url_parse(self,grade,course,source_type,page):
        if grade == "小学":
            url_grade = f"http://www.xuekeedu.com/softlist/{course}-{source_type}/index-{page}.html"
        elif grade == "初中":
            url_grade = f"http://middle.school.zxxk.com/jc/{course}-{source_type}/index-{page}.html"
        else:
            url_grade = f"http://high.school.zxxk.com/jc/{course}-{source_type}/index-{page}.html"
        print(f"正在爬取{url_grade}")
        return url_grade
        
        
# 代理地址抽取
    def proxies_initial_pick(self):
        jar = requests.cookies.RequestsCookieJar()
        cookies = "JSESSIONID=0AE7954FC784052BC1F38CC3B22254DD; _qddaz=QD.c6ygur.f1c21e.jv2ai45y"
        for cookie in cookies.split(";"):
            key,value = cookie.split("=")
            jar.set(key,value)
        response = requests.get(url = self.__API,cookies = jar,headers = self.headers,timeout = 2)
        json_data = json.loads(response.text)
        if json_data["ERRORCODE"] == "0":
            return json_data["RESULT"]
        print(json_data["RESULT"])

        
# csv文件存储模块   
    def csv_writer(self,filepath,information_generator):
        with open(f"{filepath}\data.csv","a",encoding = "UTF-8",newline = "") as f:
            writer = csv.writer(f)
            for information in information_generator:
                writer.writerow(information)
            f.close()
        
# 存储文件夹创建模块    
    def mike_dir_file_box(self,filepath):
        try:
            os.makedirs(filepath)
            with open(f"{filepath}\data.csv","a",encoding = "UTF-8",newline = "") as f:
                writer = csv.writer(f)
                writer.writerow(["课件下载地址","课件标题","上传时间","类型","大小","下载量","作者个人主页","作者",
                             "资源类型"])
        except FileExistsError as f:
            pass
    
    
    # 下载地址,课件题目,上传时间等文本提取
    def data_get_parse(self,url,proxies):
        try:
            response = self.s.get(url = url,headers = self.headers,proxies = proxies,timeout = 5)
            if response.status_code == 200:
                html_text = response.text
                pattern = re.compile(r"""(
                <a\s+class="softTitle"\s+href="(\S+)"\s+title="(.*?)"                   # 提取下载地址,课件题目
                .*?class="tim">(\d+/\d+/\d+\s+\d+:\d+:\d+)                            # 提取上传时间
                .*?class="stl">([\u4e00-\u9fa5]+)                                           # 提取文件类型
                .*?class="kb">.*?(\d+)</span>\w+</p>                                        # 提取文件大小
                .*?class="dow">.*?(\d+).*?</p>                                              # 提取下载量
                .*?class="aut">[\u4e00-\u9fa5]+.*?<a\s{1}href="(\S+)">\S+\s+\S+>(\S+)</span> # 提取作者个人主页,作者名
                .*?<a\s+class=\W+tag\s+leavel\d?\W+>\s+(\S+|\s+)\s+</a>   # 提取资源级别
                )""",re.VERBOSE|re.S)
                result_items = re.findall(pattern,html_text)
                if not result_items:
                    print(result_items)
                    return "没有匹配到对象，请检查正则"
                else:
                    information_generator = (i[1:] for i in result_items)
                    return information_generator
            else:
                return None
        except Exception as e:
            print(e)
    
# 页面最大值获取
    def top_page_get(self,url):
        try:
            response = self.s.get(url = url,headers = self.headers,timeout = 5)
            if response.status_code == 200:
                html_text = response.text
                pattern = re.compile(r"""(
                class="cur">\d+.*?(\d+)</span>
                )""",re.VERBOSE|re.S)
                result = re.search(pattern,html_text).group(2)
                return result
            else:
                return None
        except Exception as e:
            print(e)
    
    
# 数据解析
    def run(self):
        for grade,courses_sources in self.grades_dict.items():
            page = 1
            for course_chn,course_en in courses_sources[0].items():
                for kind,source_type in courses_sources[1].items():
                    filepath = f"E:\学科网\{grade}\{course_chn}\{kind}"
                    self.mike_dir_file_box(filepath)
                    page_max = self.top_page_get(self.url_parse(grade,course_en,source_type,page))
                    print(f"正在爬取{grade}-{course_chn}-{kind},最大页数为：{page_max}")
                    while True:
                        proxies_result = self.proxies_initial_pick()
                        if proxies_result:
                            for ip in proxies_result:
                                proxies = {
                                        "http":"http://" + ip["ip"]+ ":" + ip["port"],
                                        "https":"https://" + ip["ip"]+ ":" + ip["port"]
                                        }
                                while page <= int(page_max):
                                    information_generator = self.data_get_parse(self.url_parse(grade,course_en,source_type,page),
                                                                                proxies)
                                    if isinstance(information_generator,str):
                                        print(information_generator)
                                        sys.exit()
                                    elif information_generator == None:
                                        break
                                    else:
                                        self.csv_writer(filepath,information_generator)
                                        page += 1
                                        
                            time.sleep(2)
                        else:
                            break
                    winsound.Beep(self.freq,self.duration)
        sys.exit()

spider_xuekewang = SpiderXueKeWang()
spider_xuekewang.run()
