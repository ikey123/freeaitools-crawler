# 爬虫项目功能总结

## 1. 基本功能
- 接收用户提供的网站 URL
- 自动抓取网站的标题、描述和内容
- 生成网站截图和缩略图
- 使用 LLM 处理内容
- 支持多语言翻译
- 支持标签分类

## 2. 工作流程
1. 用户通过 API 发送请求，提供要爬取的 URL
   ```python
   @app.post('/site/crawl')
   async def scrape(request: URLRequest, authorization: Optional[str] = Header(None)):
       url = request.url
       tags = request.tags  # tag数组
       languages = request.languages  # 需要翻译的多语言列表

       if system_auth_secret:
           # 配置了非空的auth_secret，才验证
           validate_authorization(authorization)

       result = await website_crawler.scrape_website(url.strip(), tags, languages)

       # 若result为None,则 code="10001"，msg="处理异常，请稍后重试"
       code = 200
       msg = 'success'
       if result is None:
           code = 10001
           msg = 'fail'

       # 将数据映射到 'data' 键下
       response = {
           'code': code,
           'msg': msg,
           'data': result
       }
       return response
   ```

2. 爬虫开始工作：
   - 使用 pyppeteer 访问目标网站
   - 获取网页内容
   - 生成网站截图
   - 使用 LLM 处理内容
   ```python
   async def scrape_website(self, url, tags, languages):
       # 开始爬虫处理
       try:
           # 记录程序开始时间
           start_time = int(time.time())
           logger.info("正在处理：" + url)
           if not url.startswith('http://') and not url.startswith('https://'):
               url = 'https://' + url

           if self.browser is None:
               self.browser = await launch(headless=True,
                                           ignoreDefaultArgs=["--enable-automation"],
                                           ignoreHTTPSErrors=True,
                                           args=['--no-sandbox', '--disable-dev-shm-usage', '--disable-gpu',
                                                 '--disable-software-rasterizer', '--disable-setuid-sandbox'],
                                           handleSIGINT=False, handleSIGTERM=False, handleSIGHUP=False)

           page = await self.browser.newPage()
           # 设置用户代理
           await page.setUserAgent(random.choice(global_agent_headers))

           # 设置页面视口大小并访问具体URL
           width = 1920  # 默认宽度为 1920
           height = 1080  # 默认高度为 1080
           await page.setViewport({'width': width, 'height': height})
           try:
               await page.goto(url, {'timeout': 60000, 'waitUntil': ['load', 'networkidle2']})
           except Exception as e:
               logger.info(f'页面加载超时,不影响继续执行后续流程:{e}')
           # 获取网页内容
           origin_content = await page.content()
           soup = BeautifulSoup(origin_content, 'html.parser')

           # 通过标签名提取内容
           title = soup.title.string.strip() if soup.title else ''

           # 根据url提取域名生成name
           name = CommonUtil.get_name_by_url(url)

           # 获取网页描述
           description = ''
           meta_description = soup.find('meta', attrs={'name': 'description'})
           if meta_description:
               description = meta_description['content'].strip()

           if not description:
               meta_description = soup.find('meta', attrs={'property': 'og:description'})
               description = meta_description['content'].strip() if meta_description else ''

           logger.info(f"url:{url}, title:{title},description:{description}")

           # 生成网站截图
           image_key = oss.get_default_file_key(url)
           dimensions = await page.evaluate(f'''(width, height) => {{
               return {{
                   width: {width},
                   height: {height},
                   deviceScaleFactor: window.devicePixelRatio
               }};
           }}''', width, height)
           # 截屏并设置图片大小
           screenshot_path = './' + url.replace("https://", "").replace("http://", "").replace("/", "").replace(".", "-") + '.png'
           await page.screenshot({'path': screenshot_path, 'clip': {
               'x': 0,
               'y': 0,
               'width': dimensions['width'],
               'height': dimensions['height']
           }})
           # 上传图片，返回图片地址
           screenshot_key = oss.upload_file_to_r2(screenshot_path, image_key)

           # 生成缩略图
           thumnbail_key = oss.generate_thumbnail_image(url, image_key)

           # 抓取整个网页内容
           content = soup.get_text()

           # 使用llm工具处理content
           detail = llm.process_detail(content)
           await page.close()
   ```

3. 图片处理：
   - 生成网站截图
   - 创建缩略图
   - 上传到对象存储（如 Cloudflare R2）
   ```python
   def generate_thumbnail_image(self, url, image_key):
       # 下载图像文件
       response = self.s3.get_object(Bucket=self.S3_BUCKET_NAME, Key=image_key)
       image_data = response['Body'].read()

       # 使用Pillow库打开图像
       image = Image.open(BytesIO(image_data))

       # 将图像缩放为50%
       width, height = image.size
       new_width = int(width * 0.5)
       new_height = int(height * 0.5)
       resized_image = image.resize((new_width, new_height))

       # 创建一个BytesIO对象来保存缩略图
       thumbnail_buffer = BytesIO()
       resized_image.save(thumbnail_buffer, format='PNG')
       thumbnail_buffer.seek(0)

       # 压缩缩略图为WebP格式
       compressed_thumbnail_data = self.compress_image_to_webp(thumbnail_buffer.getvalue())

       # 将缩略图上传回S3
       thumbnail_key = self.get_default_file_key(url, is_thumbnail=True)
       self.s3.put_object(Bucket=self.S3_BUCKET_NAME, Key=thumbnail_key, Body=compressed_thumbnail_data)

       # 如果提供了自定义域名
       if self.S3_CUSTOM_DOMAIN:
           file_url = f"https://{self.S3_CUSTOM_DOMAIN}/{thumbnail_key}"
       else:
           file_url = f"{self.S3_ENDPOINT_URL}/{self.S3_BUCKET_NAME}/{thumbnail_key}"
       logger.info(f"缩略图文件URL: {file_url}")
       return file_url
   ```

## 3. 自动化特点
- 不需要手动指定要爬取的网站列表
- 通过 API 接口按需触发爬虫
- 支持异步爬取（通过 `/site/crawl_async` 接口）
- 可以配置回调 URL，爬取完成后自动通知

## 4. 数据处理
- 使用 LLM（如 Groq）处理网站内容
- 自动生成 SEO 友好的描述
- 支持多语言翻译
- 自动处理和压缩图片

## 总结
这个爬虫是一个按需工作的系统，而不是自动循环爬取固定的网站列表。它主要用于 AI 工具目录的信息更新和网站内容摘要生成。