```python
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.common.keys import Keys
from bs4 import BeautifulSoup as bs
from copy import deepcopy
import pyperclip
import time
import csv
import pandas as pd
import numpy as np
import re
from matplotlib import font_manager, rc
import matplotlib.pyplot as plt

font_name = font_manager.FontProperties(fname="c:/Windows/Fonts/malgun.ttf").get_name()
rc('font', family=font_name)

ID = ''
PW = ''
count = 0
form_str = ''
pandas_list = []
contents_list = []

def copy_input(xpath, input, driver):
    
    pyperclip.copy(input)
    driver.find_element_by_xpath(xpath).click()
#     ActionChains(driver).key_down(Keys.CONTROL).send_keys('v').key_up(Keys.CONTROL).perform() #WINDOWS
    ActionChains(driver).key_down(Keys.COMMAND).send_keys('v').key_up(Keys.CONTROL).perform() #MAC
    time.sleep(1)

def login_naver(driverpath):
#     driver = webdriver.Chrome('/usr/local/bin/chromedriver')
    driver = webdriver.Chrome(driverpath)
    driver.implicitly_wait(3)

    driver.get('https://nid.naver.com/nidlogin.login?mode=form&url=https%3A%2F%2Fwww.naver.com')

    copy_input('//*[@id="id"]', ID, driver)
    time.sleep(1)
    copy_input('//*[@id="pw"]', PW, driver)
    time.sleep(1)
    driver.find_element_by_xpath('//*[@id="log.login"]').click()
    
    return driver

def crawling(filepath, driver):
    
    cafeurl = 'https://cafe.naver.com/sssw/'
    clubid = 10625158
    menuid = 371
    userdisplay = 50
    page_range = 20
    
    
    keywords_list = ['로우 범고래', '서울', '골든로드', 'SB 시카고', '카시나 덩크 로우 블루', '카시나 덩크 넵튠 그린 민트']
    keywords_id_list = ['DD1391-100', 'DM7708-100', 'DD1391-004', 'BQ6817-600', 'CZ6501-100','CZ6501-101']
    keywords_query = ['%08%B7%CE%BF%EC+%B9%FC%B0%ED%B7%A1']
    
    for idx in range(len(keywords_list)):    
        contents_list = []
        for i in range(page_range):
            page = str(i+1)
            content_addr = 'https://cafe.naver.com/ArticleSearchList.nhn?search.clubid=' + str(clubid) + '&search.menuid=' + str(menuid) + '&search.media=0&search.searchdate=all&search.exact=&search.include=' + '&userDisplay=' + str(userdisplay) +'&search.exclude=&search.option=0&search.sortBy=date&search.searchBy=0&search.includeAll=' +'&search.query=' + keywords_query[idx] +'&search.viewtype=title' +'&search.page=' + str(page)
            
            driver.get(content_addr) # 게시판 클릭하자마자 바로 뜨는 50개 리스트
        
#             driver.execute_script('window.scrollTo(0,2700)')
#             driver.switch_to.frame('cafe_main')
#             driver.implicitly_wait(3)
#             driver.find_element_by_xpath('//*[@id="currentSearchBy"]').click()
#             driver.find_element_by_xpath('//*[@id="divSearchBy"]/ul/li[2]/a').click()
#             copy_input('//*[@id="query"]', keywords_list[idx], driver)
#             driver.find_element_by_xpath('//*[@id="main-area"]/div[7]/form/div[3]/button').click()
            
            keyword_id = keywords_id_list[idx]
        
            driver.switch_to.frame('cafe_main')
            soup = bs(driver.page_source, 'html.parser')
#             print(soup)
            soup = soup.find_all(class_='article-board result-board m-tcol-c')[0]
#             print(soup)
            datas = soup.find_all(class_='td_article')

            for data in datas:
                article_title = data.find(class_='article')
                link = article_title.get('href')
                article_title = article_title.get_text().strip()
                contents_list.append(cafeurl + link)

        count = 0
        for content in contents_list:
            driver.get(content)
            pandas_list2 =[]
        

            driver.switch_to.frame('cafe_main')
            soup = bs(driver.page_source, 'html.parser')
            date = soup.find(class_='date').text
            date = date.split(' ')[0]
            soup = soup.find_all(class_='se-text-paragraph se-text-paragraph-align-left')

            if len(soup)==0:
                soup = bs(driver.page_source, 'html.parser')
                soup = soup.find_all(class_='se-text-paragraph se-text-paragraph-align-')
            
            try:
                for p_soup in soup:
                    num = ['5','6','7'] # '판매제품명','사이즈','제품상태','희망가격'
                    form_str = p_soup.find('span').text
                    if form_str[0]=='4':
                        pandas_list2.append(keyword_id)

                #
                    if(form_str[0] in num): 
                        if form_str[0] =='5' and form_str[0] == '6' and form_str[0] == '7':
                            if(num[-2] == ' '): #빈 양식일떄
                                pass
                        elif form_str[0] == '6':
                            print(form_str.split(':')[1])
                            if'새' in form_str.split(':')[1] or '새제품' in form_str.split(':')[1] or '새상품' in form_str.split(':')[1]:
                                pandas_list2.append(0)
                            elif form_str.split(':')[1] == "": 
                                pandas_list2.append(np.nan)
                            else:
                                pandas_list2.append(1)
                        else:
                            try:
                                pandas_list2.append(form_str.split(':')[1])
                            except(IndexError, ValueError) as e:
                                print(e)
                            
            except(IndexError, ValueError) as e:
                print(e)
                
            pandas_list2.append(date)
            if len(pandas_list2)==5:
                pandas_list.append(pandas_list2)
            else:
                pass
            count = count+1
            
        break
    
    df = pd.DataFrame(pandas_list,columns=['판매제품명', '사이즈', '제품상태', '희망가격', '날짜'])
    print(df)
    df.to_csv(filepath,index=False,encoding = 'utf-8-sig', mode='w')
    
    

def convert_contents(filepath):
    contents_df = pd.read_csv(filepath)
    print()
    re_size = re.compile(r'\d{3}')
    for idx, size in enumerate(contents_df['사이즈']):
        try:
            m_size = re_size.search(size)
            converted_size = m_size.group()
            contents_df['사이즈'][idx] = converted_size
        except TypeError as T:
            contents_df['사이즈'][idx] = np.nan
        except AttributeError as A:
            contents_df['사이즈'][idx] = np.nan

    re_price = re.compile(r'\d+.?,?\d+')
    # arr = []
    for idx, price in enumerate(contents_df['희망가격']):
        try:
            m_price = re_price.search(price)
            converted_price = m_price.group()
            if '.' in converted_price:
                converted_price = int(float(converted_price)*10000)
            elif len(converted_price) < 4:
                converted_price = int(converted_price)*10000
            elif ',' in converted_price:
                converted_price = converted_price.replace(',', '')
            if type(converted_price) == int or converted_price.isdigit(): #위에 3조건 다 해도 안되는 것들을 if, else로 한번 더 묶음
                contents_df.at[idx, '희망가격'] = int(converted_price)
                #"판명하고자하는문자열".isdigit() or str.isdigit("판단하고자 하는 문자열")
                # return => True, False 
            else:
                contents_df['희망가격'][idx] = np.nan
        except TypeError as T:
            contents_df['희망가격'][idx] = np.nan
        except AttributeError as A:
            contents_df['희망가격'][idx] = np.nan
    contents_df.dropna(inplace=True)
    print(contents_df)
    
    return contents_df

def drop_outliers(data):
    q1, q3 = np.percentile(data, [25,75])
    iqr = q3 - q1
    lower_bound = q1 - (iqr * 1.5)
    upper_bound = q3 + (iqr * 1.5)

    mask = np.where((data > upper_bound) | (data < lower_bound))

#     print(q1, q3, iqr, lower_bound, upper_bound, mask)

    for idx in mask:
        data.loc[idx] = np.nan

if __name__ == '__main__':
    
#     crawling('contents.csv', login_naver(r'/usr/local/bin/chromedriver'))
    df = convert_contents('contents.csv')
    df.dropna(axis=0, inplace=True) 
    df.reset_index(drop=True, inplace=True)
    
    drop_outliers(df['희망가격'])
    df.dropna(axis=0, inplace=True) 
    df.reset_index(drop=True, inplace=True) # inplace => DATAFRAME객체 만들지 않고 df에서 처리한다는 의미..
    
    df.groupby(['날짜'])['희망가격'].agg(['mean']).plot(legend=False, figsize = (20,10),fontsize = 20)
    # df.groupby("칼럼")연속data.agg(함수)
```

    
              판매제품명  사이즈 제품상태    희망가격           날짜
    0    DD1391-100  295    1  300000  2021.11.18.
    1    DD1391-100  235    0  295000  2021.11.18.
    2    DD1391-100  265    0  290000  2021.11.18.
    3    DD1391-100  205    0  250000  2021.11.18.
    5    DD1391-100  260    0  300000  2021.11.18.
    ..          ...  ...  ...     ...          ...
    991  DD1391-100  240    0  350000  2021.09.12.
    995  DD1391-100  275    0  260000  2021.09.12.
    996  DD1391-100  280    1  290000  2021.09.12.
    997  DD1391-100  240    0  350000  2021.09.11.
    999  DD1391-100  275    1  260000  2021.09.11.
    
    [310 rows x 5 columns]
    


    
![png](output_0_1.png)
    



```python
df.dtypes # 각 열의 데이터 타입 확인
```




    판매제품명    object
    사이즈      object
    제품상태     object
    희망가격     object
    날짜       object
    dtype: object




```python
df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>판매제품명</th>
      <th>사이즈</th>
      <th>제품상태</th>
      <th>희망가격</th>
      <th>날짜</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>DD1391-100</td>
      <td>295</td>
      <td>1</td>
      <td>300000</td>
      <td>2021.11.18.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>DD1391-100</td>
      <td>235</td>
      <td>0</td>
      <td>295000</td>
      <td>2021.11.18.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>DD1391-100</td>
      <td>265</td>
      <td>0</td>
      <td>290000</td>
      <td>2021.11.18.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>DD1391-100</td>
      <td>205</td>
      <td>0</td>
      <td>250000</td>
      <td>2021.11.18.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>DD1391-100</td>
      <td>260</td>
      <td>0</td>
      <td>300000</td>
      <td>2021.11.18.</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>276</th>
      <td>DD1391-100</td>
      <td>240</td>
      <td>0</td>
      <td>350000</td>
      <td>2021.09.12.</td>
    </tr>
    <tr>
      <th>277</th>
      <td>DD1391-100</td>
      <td>275</td>
      <td>0</td>
      <td>260000</td>
      <td>2021.09.12.</td>
    </tr>
    <tr>
      <th>278</th>
      <td>DD1391-100</td>
      <td>280</td>
      <td>1</td>
      <td>290000</td>
      <td>2021.09.12.</td>
    </tr>
    <tr>
      <th>279</th>
      <td>DD1391-100</td>
      <td>240</td>
      <td>0</td>
      <td>350000</td>
      <td>2021.09.11.</td>
    </tr>
    <tr>
      <th>280</th>
      <td>DD1391-100</td>
      <td>275</td>
      <td>1</td>
      <td>260000</td>
      <td>2021.09.11.</td>
    </tr>
  </tbody>
</table>
<p>281 rows × 5 columns</p>
</div>




```python
df.groupby(['날짜'])['희망가격'].agg(['mean']).iloc[-1] #['mean'] # 전날 대비 바 그래프
#df.groupby(['날짜'])['희망가격'].agg(['mean']).plot.bar()
```




    mean    296647.058824
    Name: 2021.11.18., dtype: float64




```python
type(df.groupby(['사이즈'])['날짜'].agg(['count']))
df.groupby(['사이즈'])['날짜'].agg(['count']).plot.pie(subplots=True, figsize=(20,20),fontsize = 20, legend = False)
#df.groupby(['사이즈']).count()
```




    array([<AxesSubplot:ylabel='count'>], dtype=object)




    
![png](output_4_1.png)
    



```python
plt.legend(('평균 가격'))
df.groupby(['날짜'])['희망가격'].agg(['mean']).plot()
```


    ---------------------------------------------------------------------------

    NameError                                 Traceback (most recent call last)

    C:\Users\Public\Documents\ESTsoft\CreatorTemp/ipykernel_11296/936345695.py in <module>
    ----> 1 plt.legend(('평균 가격'))
          2 df.groupby(['날짜'])['희망가격'].agg(['mean']).plot()
    

    NameError: name 'plt' is not defined



```python
date = []
#date.append(df['날짜'][::-1])
len(df['날짜'][::-1])
for i in range(len(df['날짜'][::-1])):
    date.append(df['날짜'][::-1][i])
print(set(date))
#df['날짜'][::-1]
```

    {'2021.10.12.', '2021.11.01.', '2021.10.30.', '2021.10.16.', '2021.09.15.', '2021.10.14.', '2021.09.29.', '2021.11.02.', '2021.09.11.', '2021.11.12.', '2021.10.03.', '2021.11.06.', '2021.10.29.', '2021.11.16.', '2021.10.21.', '2021.10.19.', '2021.11.15.', '2021.10.22.', '2021.10.11.', '2021.10.18.', '2021.09.12.', '2021.10.07.', '2021.11.13.', '2021.11.07.', '2021.10.26.', '2021.10.15.', '2021.11.05.', '2021.11.10.', '2021.11.04.', '2021.09.27.', '2021.10.27.', '2021.11.11.', '2021.10.17.', '2021.11.09.', '2021.10.25.', '2021.10.13.', '2021.10.10.', '2021.10.08.', '2021.10.05.', '2021.09.28.', '2021.11.14.', '2021.09.26.', '2021.10.02.', '2021.10.24.', '2021.09.25.', '2021.10.20.', '2021.10.04.', '2021.11.03.', '2021.10.28.', '2021.10.31.', '2021.11.08.', '2021.10.09.', '2021.10.23.', '2021.11.18.', '2021.10.06.', '2021.11.17.'}
    


```python
#df.sort_values(by=['날짜'])
#df['희망가격'].value_counts().plot(x='날짜', y='희망가격')
#real_df['희망가격'].to_csv("csvs1.csv", encoding= "utf-8-sig")
#print(df.sort_values(by=['날짜']))
new_df = df.set_index('날짜')
real_df = new_df.sort_values(by=['날짜'])
real_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>판매제품명</th>
      <th>사이즈</th>
      <th>제품상태</th>
      <th>희망가격</th>
    </tr>
    <tr>
      <th>날짜</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2021.09.11.</th>
      <td>DD1391-100</td>
      <td>275</td>
      <td>1</td>
      <td>260000</td>
    </tr>
    <tr>
      <th>2021.09.11.</th>
      <td>DD1391-100</td>
      <td>240</td>
      <td>0</td>
      <td>350000</td>
    </tr>
    <tr>
      <th>2021.09.12.</th>
      <td>DD1391-100</td>
      <td>275</td>
      <td>0</td>
      <td>260000</td>
    </tr>
    <tr>
      <th>2021.09.12.</th>
      <td>DD1391-100</td>
      <td>280</td>
      <td>1</td>
      <td>290000</td>
    </tr>
    <tr>
      <th>2021.09.12.</th>
      <td>DD1391-100</td>
      <td>240</td>
      <td>0</td>
      <td>350000</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>2021.11.18.</th>
      <td>DD1391-100</td>
      <td>250</td>
      <td>0</td>
      <td>280000</td>
    </tr>
    <tr>
      <th>2021.11.18.</th>
      <td>DD1391-100</td>
      <td>280</td>
      <td>1</td>
      <td>319000</td>
    </tr>
    <tr>
      <th>2021.11.18.</th>
      <td>DD1391-100</td>
      <td>220</td>
      <td>0</td>
      <td>290000</td>
    </tr>
    <tr>
      <th>2021.11.18.</th>
      <td>DD1391-100</td>
      <td>280</td>
      <td>0</td>
      <td>310000</td>
    </tr>
    <tr>
      <th>2021.11.18.</th>
      <td>DD1391-100</td>
      <td>295</td>
      <td>1</td>
      <td>300000</td>
    </tr>
  </tbody>
</table>
<p>310 rows × 4 columns</p>
</div>




```python
#real_df.plot(x='날짜', y='희망가격')
real_df['희망가격'] = real_df['희망가격'].astype(float)
```


    ---------------------------------------------------------------------------

    ValueError                                Traceback (most recent call last)

    C:\Users\Public\Documents\ESTsoft\CreatorTemp/ipykernel_21444/178906551.py in <module>
          1 #real_df.plot(x='날짜', y='희망가격')
    ----> 2 real_df['희망가격'] = real_df['희망가격'].astype(float)
    

    ~\anaconda3\lib\site-packages\pandas\core\generic.py in astype(self, dtype, copy, errors)
       5813         else:
       5814             # else, only a single dtype is given
    -> 5815             new_data = self._mgr.astype(dtype=dtype, copy=copy, errors=errors)
       5816             return self._constructor(new_data).__finalize__(self, method="astype")
       5817 
    

    ~\anaconda3\lib\site-packages\pandas\core\internals\managers.py in astype(self, dtype, copy, errors)
        416 
        417     def astype(self: T, dtype, copy: bool = False, errors: str = "raise") -> T:
    --> 418         return self.apply("astype", dtype=dtype, copy=copy, errors=errors)
        419 
        420     def convert(
    

    ~\anaconda3\lib\site-packages\pandas\core\internals\managers.py in apply(self, f, align_keys, ignore_failures, **kwargs)
        325                     applied = b.apply(f, **kwargs)
        326                 else:
    --> 327                     applied = getattr(b, f)(**kwargs)
        328             except (TypeError, NotImplementedError):
        329                 if not ignore_failures:
    

    ~\anaconda3\lib\site-packages\pandas\core\internals\blocks.py in astype(self, dtype, copy, errors)
        590         values = self.values
        591 
    --> 592         new_values = astype_array_safe(values, dtype, copy=copy, errors=errors)
        593 
        594         new_values = maybe_coerce_values(new_values)
    

    ~\anaconda3\lib\site-packages\pandas\core\dtypes\cast.py in astype_array_safe(values, dtype, copy, errors)
       1307 
       1308     try:
    -> 1309         new_values = astype_array(values, dtype, copy=copy)
       1310     except (ValueError, TypeError):
       1311         # e.g. astype_nansafe can fail on object-dtype of strings
    

    ~\anaconda3\lib\site-packages\pandas\core\dtypes\cast.py in astype_array(values, dtype, copy)
       1255 
       1256     else:
    -> 1257         values = astype_nansafe(values, dtype, copy=copy)
       1258 
       1259     # in pandas we don't store numpy str dtypes, so convert to object
    

    ~\anaconda3\lib\site-packages\pandas\core\dtypes\cast.py in astype_nansafe(arr, dtype, copy, skipna)
       1199     if copy or is_object_dtype(arr.dtype) or is_object_dtype(dtype):
       1200         # Explicit copy, or required since NumPy can't view from / to object.
    -> 1201         return arr.astype(dtype, copy=True)
       1202 
       1203     return arr.astype(dtype, copy=copy)
    

    ValueError: could not convert string to float: ' 26만9천'



```python
real_df['희망가격'].str.contains(str)
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    C:\Users\Public\Documents\ESTsoft\CreatorTemp/ipykernel_21444/1758154325.py in <module>
    ----> 1 real_df['희망가격'].str.contains(str)
    

    ~\anaconda3\lib\site-packages\pandas\core\strings\accessor.py in wrapper(self, *args, **kwargs)
        114                 )
        115                 raise TypeError(msg)
    --> 116             return func(self, *args, **kwargs)
        117 
        118         wrapper.__name__ = func_name
    

    ~\anaconda3\lib\site-packages\pandas\core\strings\accessor.py in contains(self, pat, case, flags, na, regex)
       1151         dtype: bool
       1152         """
    -> 1153         if regex and re.compile(pat).groups:
       1154             warnings.warn(
       1155                 "This pattern has match groups. To actually get the "
    

    ~\anaconda3\lib\re.py in compile(pattern, flags)
        250 def compile(pattern, flags=0):
        251     "Compile a regular expression pattern, returning a Pattern object."
    --> 252     return _compile(pattern, flags)
        253 
        254 def purge():
    

    ~\anaconda3\lib\re.py in _compile(pattern, flags)
        301         return pattern
        302     if not sre_compile.isstring(pattern):
    --> 303         raise TypeError("first argument must be string or compiled pattern")
        304     p = sre_compile.compile(pattern, flags)
        305     if not (flags & DEBUG):
    

    TypeError: first argument must be string or compiled pattern



```python
real_df['희망가격'].dropna(axis=0)
```




    0      300000
    1      295000
    2      290000
    3      250000
    5      300000
            ...  
    991    350000
    995    260000
    996    290000
    997    350000
    999    260000
    Name: 희망가격, Length: 312, dtype: object




```python
real_df['희망가격'] = real_df['희망가격'].astype(float)
```


    ---------------------------------------------------------------------------

    ValueError                                Traceback (most recent call last)

    C:\Users\Public\Documents\ESTsoft\CreatorTemp/ipykernel_21444/4107160396.py in <module>
    ----> 1 real_df['희망가격'] = real_df['희망가격'].astype(float)
    

    ~\anaconda3\lib\site-packages\pandas\core\generic.py in astype(self, dtype, copy, errors)
       5813         else:
       5814             # else, only a single dtype is given
    -> 5815             new_data = self._mgr.astype(dtype=dtype, copy=copy, errors=errors)
       5816             return self._constructor(new_data).__finalize__(self, method="astype")
       5817 
    

    ~\anaconda3\lib\site-packages\pandas\core\internals\managers.py in astype(self, dtype, copy, errors)
        416 
        417     def astype(self: T, dtype, copy: bool = False, errors: str = "raise") -> T:
    --> 418         return self.apply("astype", dtype=dtype, copy=copy, errors=errors)
        419 
        420     def convert(
    

    ~\anaconda3\lib\site-packages\pandas\core\internals\managers.py in apply(self, f, align_keys, ignore_failures, **kwargs)
        325                     applied = b.apply(f, **kwargs)
        326                 else:
    --> 327                     applied = getattr(b, f)(**kwargs)
        328             except (TypeError, NotImplementedError):
        329                 if not ignore_failures:
    

    ~\anaconda3\lib\site-packages\pandas\core\internals\blocks.py in astype(self, dtype, copy, errors)
        590         values = self.values
        591 
    --> 592         new_values = astype_array_safe(values, dtype, copy=copy, errors=errors)
        593 
        594         new_values = maybe_coerce_values(new_values)
    

    ~\anaconda3\lib\site-packages\pandas\core\dtypes\cast.py in astype_array_safe(values, dtype, copy, errors)
       1307 
       1308     try:
    -> 1309         new_values = astype_array(values, dtype, copy=copy)
       1310     except (ValueError, TypeError):
       1311         # e.g. astype_nansafe can fail on object-dtype of strings
    

    ~\anaconda3\lib\site-packages\pandas\core\dtypes\cast.py in astype_array(values, dtype, copy)
       1255 
       1256     else:
    -> 1257         values = astype_nansafe(values, dtype, copy=copy)
       1258 
       1259     # in pandas we don't store numpy str dtypes, so convert to object
    

    ~\anaconda3\lib\site-packages\pandas\core\dtypes\cast.py in astype_nansafe(arr, dtype, copy, skipna)
       1199     if copy or is_object_dtype(arr.dtype) or is_object_dtype(dtype):
       1200         # Explicit copy, or required since NumPy can't view from / to object.
    -> 1201         return arr.astype(dtype, copy=True)
       1202 
       1203     return arr.astype(dtype, copy=copy)
    

    ValueError: could not convert string to float: ' 26만9천'



```python
real_df.plot(x='날짜', y='희망가격')
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    C:\Users\Public\Documents\ESTsoft\CreatorTemp/ipykernel_22348/3508921356.py in <module>
    ----> 1 real_df.plot(x='날짜', y='희망가격')
    

    ~\anaconda3\lib\site-packages\pandas\plotting\_core.py in __call__(self, *args, **kwargs)
        970                     data.columns = label_name
        971 
    --> 972         return plot_backend.plot(data, kind=kind, **kwargs)
        973 
        974     __call__.__doc__ = __doc__
    

    ~\anaconda3\lib\site-packages\pandas\plotting\_matplotlib\__init__.py in plot(data, kind, **kwargs)
         69             kwargs["ax"] = getattr(ax, "left_ax", ax)
         70     plot_obj = PLOT_CLASSES[kind](data, **kwargs)
    ---> 71     plot_obj.generate()
         72     plot_obj.draw()
         73     return plot_obj.result
    

    ~\anaconda3\lib\site-packages\pandas\plotting\_matplotlib\core.py in generate(self)
        284     def generate(self):
        285         self._args_adjust()
    --> 286         self._compute_plot_data()
        287         self._setup_subplots()
        288         self._make_plot()
    

    ~\anaconda3\lib\site-packages\pandas\plotting\_matplotlib\core.py in _compute_plot_data(self)
        451         # no non-numeric frames or series allowed
        452         if is_empty:
    --> 453             raise TypeError("no numeric data to plot")
        454 
        455         self.data = numeric_data.apply(self._convert_to_ndarray)
    

    TypeError: no numeric data to plot



```python
li1 = []
li2 = []
li1 = set(df['날짜'])
characters ="."

li2 = [x for x in li1]
print(li2)

for i in range(len(li2)):
    li2[i] =  li2[i].replace(characters[0],"") # 날짜 set -> sort하려고
print(li2)

list = [int(x) for x in li2]
list.sort()
print(list)


```

    ['2021.10.12.', '2021.11.01.', '2021.10.30.', '2021.10.16.', '2021.09.15.', '2021.10.14.', '2021.09.29.', '2021.11.02.', '2021.09.11.', '2021.11.12.', '2021.10.03.', '2021.11.06.', '2021.10.29.', '2021.11.16.', '2021.10.21.', '2021.10.19.', '2021.11.15.', '2021.10.22.', '2021.10.11.', '2021.10.18.', '2021.09.12.', '2021.10.07.', '2021.11.13.', '2021.11.07.', '2021.10.26.', '2021.10.15.', '2021.11.05.', '2021.11.10.', '2021.11.04.', '2021.09.27.', '2021.10.27.', '2021.11.11.', '2021.10.17.', '2021.11.09.', '2021.10.25.', '2021.10.13.', '2021.10.10.', '2021.10.08.', '2021.10.05.', '2021.09.28.', '2021.11.14.', '2021.09.26.', '2021.10.02.', '2021.10.24.', '2021.09.25.', '2021.10.20.', '2021.10.04.', '2021.11.03.', '2021.10.28.', '2021.10.31.', '2021.11.08.', '2021.10.09.', '2021.10.23.', '2021.11.18.', '2021.10.06.', '2021.11.17.']
    ['20211012', '20211101', '20211030', '20211016', '20210915', '20211014', '20210929', '20211102', '20210911', '20211112', '20211003', '20211106', '20211029', '20211116', '20211021', '20211019', '20211115', '20211022', '20211011', '20211018', '20210912', '20211007', '20211113', '20211107', '20211026', '20211015', '20211105', '20211110', '20211104', '20210927', '20211027', '20211111', '20211017', '20211109', '20211025', '20211013', '20211010', '20211008', '20211005', '20210928', '20211114', '20210926', '20211002', '20211024', '20210925', '20211020', '20211004', '20211103', '20211028', '20211031', '20211108', '20211009', '20211023', '20211118', '20211006', '20211117']
    [20210911, 20210912, 20210915, 20210925, 20210926, 20210927, 20210928, 20210929, 20211002, 20211003, 20211004, 20211005, 20211006, 20211007, 20211008, 20211009, 20211010, 20211011, 20211012, 20211013, 20211014, 20211015, 20211016, 20211017, 20211018, 20211019, 20211020, 20211021, 20211022, 20211023, 20211024, 20211025, 20211026, 20211027, 20211028, 20211029, 20211030, 20211031, 20211101, 20211102, 20211103, 20211104, 20211105, 20211106, 20211107, 20211108, 20211109, 20211110, 20211111, 20211112, 20211113, 20211114, 20211115, 20211116, 20211117, 20211118]
    
