#### python变量名命名规则:

1. 变量名通常由字母,数字,下划线组成
2. 数字不能作为变量名开头
3. 不能以python中的关键字、包名和py文件名命名
4. 变量名要有意义
5. 不要用汉字和拼音去命名
6. 变量名要区分大小写
7. 推荐使用驼峰型(GuessAge或guessAge)和**下划线**(guess_age)来命名
8. 常量通常使用大写来定义



#### 函数名命名规则：

1. 一般采用动词命名，或动词加名词，做到见名知意。
2. 不能以python中的关键字、包名和py文件名命名



#### 关键字

```python
['False', 'None', 'True', '__peg_parser__', 'and', 'as', 'assert', 'async', 'await', 'break', 'class', 'continue', 'def', 'del', 'elif', 'else', 'except', 'finally', 'for', 'from', 'global', 'if', 'import', 'in', 'is', 'lambda', 'nonlocal', 'not', 'or', 'pass', 'raise', 'return', 'try', 'while', 'with', 'yield']

```



#### try except异常处理

​	异常捕获需指定异常类型，或者输出异常信息添加注释



#### 函数参数类型约束

1. 易于重构

   类型提示可以在重构时，更好得帮助我们定位类的位置。

2. 易于理解代码

   在调用函数时就可以告诉你需要传递哪些参数类型；以及需要扩展/修改函数时，也会告诉你输入和输出所需要的数据类型

3. 验证运行数据

   实现类型提示的规范方式

![截屏2022-08-12 01.08.17](https://tva1.sinaimg.cn/large/e6c9d24egy1h53bpu43ohj21wg0mygpu.jpg)

#### 函数内代码行数限制

​	30行以内













```python
def deal(self, data):
        if isinstance(data, str):
            try:
                data = data.strip()
                return data
            except:
                return None
        if data:
            temp_list = []
            for i in data:
                try:
                    temp = i.strip()
                    if temp:
                        temp_list.append(temp)
                except:
                    continue
            if temp_list:
                return temp_list
            else:
                return None
        else:
            return None

```



![截屏2022-08-12 01.38.07](https://tva1.sinaimg.cn/large/e6c9d24egy1h53clggk5zj21e60oiwi5.jpg)





```python
def authors(self, response,article_id,article_url_id):
        li_list=response.xpath('//ol[@class="contributor-list"]/li')
        authors_num=0
        for li in li_list:  #遍历所有作者
            authors_id=get_snowflake()
            name=self.deal(li.xpath('./span[@class="name"]/text()').get())
            a_list=li.xpath('./a')
            temp_list = []  # 作者序号信息
            temp_all_dict = {}
            for a in a_list:
                num=''.join(self.deal(a.xpath('.//text()').getall()))
                num_info=response.xpath(f'//ol[@class="affiliation-list"]//sup[text()="{num}"]/following-sibling::text()').get()
                temp_list.append(self.deal(num_info))
            temp_all_dict['name']=name
            temp_all_dict['info']=temp_list
            # temp_all_list.append(temp_all_dict)
            self.mysql_db.add(f'insert into BatchPreprintAuthor set id={authors_id},author_num="{authors_num}",affiliations="{temp_all_dict["info"]}",author_name="{temp_all_dict["name"]}"')  #插入作者表
            self.mysql_db.add(f'insert into BatchPreprintArticleAuthor set id={get_snowflake()},article_id={article_id},author_id={authors_id},article_url_id={article_url_id}') #插入关联表
            authors_num+=1
        # return temp_all_list

```

```python
def parse_b(self, request, response):
        task = request.task
        all_data_list = response.xpath('//div[@class="highwire-list"]/ul/li')
        for data in all_data_list:  #遍历每条产品
            product_dict = {}
            product_dict['article_url_id']=task.get('id')
            product_dict['article_url'] = task.get('url')
            product_dict['url'] = data.xpath('.//span[@class="highwire-cite-title"]/a/@href').get()
            if self.find_url(product_dict['url']):
                return False  # 如果url存在则停止程序
            else:
                product_dict['doi'] = data.xpath('//span[@class="doi_label"]/parent::span/text()').get()
                yield feapder.Request(url=product_dict['url'],product_dict=product_dict, callback=self.parse_c)

    def parse_c(self, request, response):
        response.code = 'utf-8'
        product_dict = request.product_dict
        product_dict['url'] = request.url
        product_dict['id'] = get_snowflake()  # 文章基本信息id
        self.source(product_dict['id'], response.text, product_dict['url'])  # 保存源码
        product_dict['pdf_url'] = self.deal(response.xpath("//meta[@name='citation_pdf_url']/@content").get())
        product_dict['xml_url'] = product_dict['pdf_url'].replace('.full.pdf', '.source.xml')
        product_dict['title'] = self.deal(response.xpath('//h1[@id="page-title"]/text()').get())
        product_dict['doi'] = self.deal(response.xpath('//h1/following-sibling::*//span[text()="doi:"]/following-sibling::text()').get())
        product_dict['abstract'] = self.abstract(response)
        product_dict['date'] = self.deal(response.xpath('//div[contains(text(),"Posted")]/text()').get())
        article_info_url = request.url + '.article-info'  # history信息页面,获取doi的变更信息
        yield feapder.Request(url=article_info_url,product_dict=product_dict, callback=self.parse_d)

```

```python
def parse(self, request, response):
        if response.status_code == 200:
            response.text = text_clear.remove_tag(response.text)
            task = request.task
            product_data = {}
            product_data.update(task)
            soup = response.bs4()
            name = soup.find('div', id="product-title").find('h1').text.strip()
            table_node = soup.find('table', id="table-product")
            table_node.find('tr', class_="last").decompose()
            df = pd.read_html(str(table_node), keep_default_na=False)[0]
            order_info = df.to_dict(orient="record")
            product_data['name'] = name
            product_data['order_info'] = order_info

            product_detail = []
            detail_nodes = soup.find('div', class_="detail-container").find_all('div', class_="section")
            for node in detail_nodes:
                title = node.find('h4').text.strip()
                if '<strong' in str(node):
                    for ele in node.find_all('strong'):
                        key = ele.text.strip()
                        for _ in ele.next_siblings:
                            if _.name:
                                if _.name == 'strong':
                                    break
                                elif _.name == 'span':
                                    value = _.text.strip()
                                    product_detail.append({title: {key: value}})
                else:
                    temp_value = []
                    for ele in node.find_all('p'):
                        temp_value.append(ele.text.strip())
                    product_detail.append({title: temp_value})
            product_data['product_detail'] = product_detail
            print(product_data)
            product_data = text_clear.dict_add_tag(product_data)
            db_mongo.add('Everestbiotech', product_data, replace=True)

```

http://code.deepbiogroup.com/data/gerbu_spider/-/blob/master/spiders/gerbu_parser.py

```python
def parse(self, request, response):
        if response.status_code == 200:
            response.text = text_clear.remove_tag(response.text)
            task = request.task
            product_data = {}
            product_data.update(task)
            soup = response.bs4()
            path = '>>'.join([_.text.strip() for _ in soup.find('ol', class_="breadcrumb").find_all('a')])
            name = soup.find('h1').text.strip()
            try:
                description = soup.find('div', itemprop="description").text.strip()
            except:
                description = None
            price = soup.find('div', class_="price").text.strip()

            df = pd.read_html(str(soup.find('div', class_="product-matrix").find('table')))[0]
            order_info = df.to_dict(orient='record')

            df2 = pd.read_html(str(soup.find('div', class_="product-attributes").find('table')))[0]
            temp = df2.set_index(0).to_dict()[1]

            product_data['path'] = path
            product_data['name'] = name
            product_data['description'] = description
            product_data['price'] = price
            for ele in soup.find('ul', class_="list-unstyled").find_all('li'):
                key = [i for i in ele if i.name][0].text.strip()
                value = [i for i in ele if i.name][1].text.strip()
                product_data[key] = value
            product_data['order_info'] = order_info
            product_data.update(temp)

            print(product_data)
            product_data = text_clear.dict_add_tag(product_data)
            db_mongo.add('Gerbu', product_data, replace=True)


```

```python
def parse(self, request, response):
        if response.status_code == 200:
            task = request.task
            response.text = text_clear.remove_tag(response.text)
            soup = response.bs4()
            product_data = {}
            product_data.update(task)
            path = '>>'.join([_.text.strip() for _ in soup.find('section',class_="breadcrumbs").find_all('a')])
            product_type = soup.find('ul',class_="tabs").find('li',class_="active").text.strip()
            data_node = soup.find('ul',class_="productItem")
            name = data_node.find('h2',itemprop="name").text.strip()
            description = data_node.find('div',itemprop="description").text.strip()
            catalog_no = data_node.find('li',class_="catalogNumber").text.strip()
            order_info = [_.text.strip() for _ in data_node.find('li',class_="pricing").find_all('div',class_="customRadio")]
            quantity = soup.find('span',class_="shippmentDetails").text.strip()
            product_data['path'] = path
            product_data['product_type'] = product_type
            product_data['name'] = name
            product_data['description'] = description
            product_data['catalog_no'] = catalog_no
            product_data['order_info'] = order_info
            product_data['quantity'] = quantity

            main_node = soup.find('div',class_="productDetails")
            for ele in main_node.find_all('h3'):
                key = ele.text.strip()
                for node in ele.next_siblings:
                    if node.name:
                        if node.name == 'h3':
                            break
                        value = node.text.strip()
                        product_data[key] = value

            product_data = text_clear.dict_add_tag(product_data)
            print(product_data)
            db_mongo.add('Prospecbio', product_data)



```

```python
def parse(self, request, response):
        if response.status_code == 200:
            text_clear = TextCleaner()
            response.text = text_clear.remove_tag(response.text)
            task = request.task
            _id = task['_id']
            info_url = task['info_url'].decode(encoding='utf-8')
            soup = response.bs4()
            product_data = {}
            path = [_ for _ in soup.find_all('nav',class_='navbar')][1].find('a').text.strip()
            name = soup.find('h1').text.strip()
            catalog_no = soup.find('span',id="partnos").text.strip()

            product_data['_id'] = _id
            product_data['info_url'] = info_url
            product_data['name'] = name
            product_data['path'] = path
            product_data['catalog_no'] = catalog_no


            data_node = soup.find('div',id = "content")
            for node in data_node.find_all('h2'):
                title = node.text.strip()
                value = []
                for ele in node.next_siblings:
                    if ele.name:
                        if ele.name == 'h2':
                            break
                        elif ele.name == 'p':
                            second_node = [_ for _ in ele if _.name]
                            if len(second_node) > 0:
                                if second_node[0].name == 'ul':
                                    key = ele.text.strip()
                                    second_value = [_.text.strip() for _ in ele.next_sibling.next_sibling.find_all('li')]
                                    value.append({key:second_value})
                                else:
                                    value.append(ele.text.strip())
                            else:
                                value.append(ele.text.strip())
                        elif ele.name == 'div':
                            if '<table' in str(ele):
                                value.append(pd.read_html(str(ele),keep_default_na=False)[0].to_dict(orient="record"))
                            else:
                                value.append(ele.text.strip())
                        elif ele.name == 'h4':
                            key = ele.text.strip()

                            if '<ul' in str(ele):
                                second_value = {}
                                for _ in [_ for _ in ele.next_siblings if _.name and _.name == 'ul'][0].find_all('li'):
                                    temp_value = _.find('span').text.strip()
                                    _.span.decompose()
                                    temp_key = _.text.strip()
                                    second_value[temp_key] = temp_value
                                value.append({key:second_value})
                            elif '<ol' in str(ele):
                                second_value = {}
                                for _ in [_ for _ in ele.next_siblings if _.name and _.name == 'ol'][0].find_all('li'):
                                    temp_value = _.find('span').text.strip()
                                    _.span.decompose()
                                    temp_key = _.text.strip()
                                    second_value[temp_key] = temp_value
                                value.append({key: second_value})
                            elif '<table' in str(ele):
                                second_value = pd.read_html(str(ele),keep_default_na=False)[0].to_dict(orient = "record")
                                value.append({key:second_value})
                        elif ele.name == 'ul':
                            # temp_data = {}
                            # for _ in ele.find_all('li'):
                            #     data = _.text.strip()
                            #     temp_data[data.split(':')[0]] = data.split(':')[1]

                            # value.append(temp_data)
                            value.append([_.text.strip() for _ in ele.find_all('li')])
                product_data[title] = value
            print(product_data)
            product_data = text_clear.dict_add_tag(product_data)
            db_mongo.add('Licor', product_data)

```

```python
def parse(self, request, response):
        if response.status_code == 200:
            task = request.task

            text_clear = TextCleaner()
            response.text = text_clear.remove_tag(response.text)

            _id = task['_id']
            info_url = task['info_url']

            soup = response.bs4()

            product_data = {}

            product_data['_id'] = _id
            product_data['info_url'] = info_url.decode(encoding='utf-8')

            path = '>>'.join([_.get_text().strip() for _ in soup.find('div',class_='breadcrumbs').find('ul').find_all('a')])
            product_data['path'] = path
            product_name = soup.find('h1').get_text().strip()
            product_data['product_name'] = product_name
            descriiption = soup.find('div',itemprop='description').get_text().strip()
            product_data['descriiption'] = descriiption

            option_node = soup.find('div',class_="product-options-wrapper").find('script',type="text/x-magento-init")
            option = json.loads(option_node.get_text().strip())
            skus = option['#product_addtocart_form']['configurable']['spConfig']['skus']
            # print(skus)
            attributes = option['#product_addtocart_form']['configurable']['spConfig']['attributes']
            # print(attributes)
            descriptions = option['#product_addtocart_form']['configurable']['spConfig']['descriptions']
            # print(descriptions)
            index = option['#product_addtocart_form']['configurable']['spConfig']['index']
            # print(index)

            option_data = []
            for key,value in index.items():
                catalog_no = skus[key]
                description = descriptions[key]
                temp_dict = {'catalog_no':catalog_no,'description':description}
                for k,v in value.items():

                    label = attributes[k]['label']

                    for i in attributes[k]['options']:

                        if i['id'] == v:
                            label_value = i['label']
                            temp_dict[label] = label_value

                # print(temp_dict)
                option_data.append(temp_dict)
            product_data['option_data'] = option_data


            for node in soup.find('div',class_="product data items"):
                if node.name:
                    # section-overview
                    overview = {}
                    if node.get('id') == 'section-overview':

                        head_node = node.find('h2')
                        head = head_node.get_text().strip()
                        overview['head'] = head
                        value_nodes = [_ for _ in node.find('div',class_="row").children if _.name == 'div']
                        desc_node = value_nodes[0]
                        description = desc_node.find('div',class_="description").get_text().strip()
                        overview['description'] = description
                        attr_node = value_nodes[1].find('div',class_="attribute-list")
                        attribute = {}
                        for attr in attr_node.children:
                            if attr.name:
                                key = attr.find('div',class_="header").get_text().strip()
                                value = attr.find('div',class_="value").get_text().strip()
                                attribute[key] = value
                        overview['attribute'] = attribute
                        # print(overview)
                        product_data['overview'] = overview


                    # section-product-use
                    if node.get('id') == 'section-product-use':
                        head = node.find('h2').get_text().strip()
                        value_node = node.find('div',id="product-use")
                        desc = value_node.find('p',class_="description").get_text().strip()
                        table_node = value_node.find('div',class_="table")
                        table_titles = [_.get_text().strip() for _ in table_node.find('div',class_="thead").find_all('div',class_="th")]
                        # print(table_titles)
                        keys = []
                        values = []
                        for tr_node in table_node.find('div',class_="tbody").children:

                            if tr_node.name and tr_node.get('class').__contains__('tr'):
                                for td_node in tr_node.children:
                                    if td_node.name and td_node.get('class').__contains__('td'):
                                        if td_node.get('class').__contains__('workflow-name'):
                                            key = td_node.find('a').get_text().strip()
                                            # print(key)
                                            keys.append(key)
                                        elif td_node.get('class').__contains__('steps'):
                                            temp_value = []
                                            for _ in td_node.find_all('div',class_="step"):
                                                if _.get('class').__contains__('active'):
                                                    value = _.get_text().strip()
                                                    temp_value.append(value)
                                                # else:
                                                #     value = [_.get_text().strip(), 0]
                                                #     temp_value.append(value)
                                            values.append(temp_value)

                        application = {k: v for k, v in zip(keys, values)}
                        # print(application)
                        product_data['application'] = application
            print(product_data)

            product_data = text_clear.dict_add_tag(product_data)
            db_mongo.add('Stemcell', product_data)


```

http://code.deepbiogroup.com/data/chemodex_spider/-/blob/master/spiders/chemodex_spider.py

http://code.deepbiogroup.com/data/nordicmubio_spider/-/blob/master/spiders/nordicmubio_spider.py

http://code.deepbiogroup.com/data/pall_spider/-/blob/master/spiders/pall_shop_spider.py

http://code.deepbiogroup.com/data/stressmarq_spider/-/blob/master/spiders/stressmarq.py

http://code.deepbiogroup.com/data/lethub_spider/-/blob/master/spiders/letpubspider.py

冗余代码

http://code.deepbiogroup.com/data/pmc_article_spider