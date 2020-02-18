### 快速用法（配合sed/awk/grep）

```javascript
$curl http: //mydomain.net
```

#### 下载保存

```javascript
$curl http://mydomain.net > index.html
$curl -o index.html http://mydomain.net
$curl -O http://mydomain.net/target.tar.gz
```

#### GET

```javascript
$curl http://www.yahoo.com/login.cgi?user=nickname&password=12345
```

#### POST

```javascript
$curl -d "user=nickname&password=12345" http://www.yahoo.com/login.cgi
```

#### POST 文件

```javascript
$curl -F upload= $localfile  -F $btn_name=$btn_value http://mydomain.net/~zzh/up_file.cgi
```

#### 通过代理

```javascript
$curl -x 123.45.67.89:1080 -o page.html http://mydomain.net
```

#### 保存cookie

```javascript
$curl -x 123.45.67.89:1080 -o page1.html -D cookie0001.txt http://mydomain.net
```

#### 使用cookie

```javascript
$curl -x 123.45.67.89:1080 -o page1.html -D cookie0002.txt -b cookie0001.txt http://mydomain.net
```

#### 模仿浏览器

```javascript
$curl -A "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)" -x123.45.67.89:1080 -o page.html -D cookie0001.txt http://mydomain.net
```

#### 伪造referer

```javascript
$curl -A "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)" -x123.45.67.89:1080 -e"mail.yahoo.com" -o page.html -D cookie0001.txt http://mydomain.net
```

### 高级下载功能

#### 循环下载

```javascript
$curl -O http://mydomain.net/~zzh/screen[1-10].JPG
```

#### 循环（匹配）下载

```javascript
$curl -O http://mydomain.net/~{zzh,nick}/[001-201].JPG  # >like zzh/001.JPG
```

#### 循环（引用）下载

```javascript
$curl -o #2_#1.jpg http://mydomain.net/~{zzh,nick}/[001-201].JPG # like >001_zzh.jpg
```

#### 断点续传

```javascript
$curl -c -O http://mydomain.net/~zzh/screen1.JPG
```

#### 分块下载

```javascript
$curl -r  0 -10240  -o "zhao.part1"  http://mydomain.net/~zzh/zhao1.mp3 &\
$curl -r 10241 -20480  -o "zhao.part1"  http://mydomain.net/~zzh/zhao1.mp3 &\
$curl -r 20481 -40960  -o "zhao.part1"  http://mydomain.net/~zzh/zhao1.mp3 &\
$curl -r 40961 - -o  "zhao.part1"  http://mydomain.net/~zzh/zhao1.mp3
...
$cat zhao.part* > zhao.mp3
```

