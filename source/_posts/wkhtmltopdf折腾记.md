---
title: wkhtmltopdf折腾记
date: 2016-12-06 11:46:58
tags: wkhtmltopdf
---



## What for?

PRD: 打印条款以供用户下载。

After googling around,there are two possiable options:

* [iText](http://itextpdf.com/). 以前一般都用这个来做PDF的生成，但试用了一下感觉很是难用！ 

* [wkhtmltopdf](http://wkhtmltopdf.org/). 一个单独的程序，命令行可直接使用用，相对(也有一些坑)来说简单易用。官方介绍:

  > `wkhtmltopdf` and `wkhtmltoimage` are open source (LGPLv3) command line tools to render HTML into PDF and various image formats using the Qt WebKit rendering engine. These run entirely "headless" and do not require a display or display service.




<!-- more -->

## 开始捣鼓

### 本地环境

本地mac,官网下了一个pkg包，安装后终端直接:

```shell
wkhtmltopdf http://baidu.com baidu.pdf
```

yeah,it works! 不错，以为就行了，结果各种问题，字体太小，页面也感觉缩小了很多，起初以为CSS的问题，调了一下没什么卵用。于是再次Google半天，发现了这么一篇文章:[wkhtmltopdf-font-and-sizing-issues](http://blog.gluga.com/2012/05/wkhtmltopdf-font-and-sizing-issues.html), 使用了该文章中所说的`--disable-smart-shrinking` 页面缩小的问题得到解决。 但还有一个问题，页面未分页，整个诺达的HTML页面完全打成了一页，SO上看到有人说以下方法可以解决:

```css
div {
  page-break-before: always;
}

body {
  line-height: 1em;
  text-rendering: geometricPrecision;
}
```

加入style之后效果得以缓解。以为至此就算折腾完了，no way, 继续。



### 服务器环境

服务器centos, yum:

```shell
yum install wkhtmltopdf
```

兴高采烈地试一把:

```shell
wkhtmltopdf http://baidu.com baidu.pdf
```

oops, error: `can not connect to X server`.

尝试Google大法，SO有个回答：http://stackoverflow.com/a/9685072/1388881， 照着试了一下，结果还是无果。 又Google了大半天发现很多直接下载官网的安装包就没有问题，于是：

```shell
wget http://download.gna.org/wkhtmltopdf/0.12/0.12.4/wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
tar xf wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
cd wkhtmltox/bin
cp wkhtmltopdf /bin
```
再次尝试`wkhtmltopdf http://baidu.com baidu.pdf`, and it works! 只想感叹一句，能活到今天真是💗大啊!



## Code

一开始使用的是[Java WkHtmlToPdf Wrapper](https://github.com/jhonnymertz/java-wkhtmltopdf-wrapper), 但在使用过程中偶尔会出现`HostNotFoundError`. 所以放弃了。看了下源码不是很多代码，自己写了个调用`wkhtmltopdf`。



```java
public class PDFService {
    private final Logger logger = LoggerFactory.getLogger(PDFService.class);
    private Random random = new Random();

    public byte[] html2pdf(String srcUrl) {
        String tmp = System.getProperty("java.io.tmpdir");
        String filename = System.currentTimeMillis() + random.nextInt(1000) + "";
        return html2pdf(srcUrl, tmp + "/" + filename);
    }

    private byte[] html2pdf(String src, String dest) {

        String space = " ";

        String wkhtmltopdf = findExecutable();
        if (StringUtils.isEmpty(wkhtmltopdf)) {
            logger.error("no wkhtmltopdf found!");
            throw new RuntimeException("html转换pdf出错了");
        }

        File file = new File(dest);
        File parent = file.getParentFile();
        if (!parent.exists()) {
            boolean dirsCreation = parent.mkdirs();
            logger.info("create dir for new file,{}", dirsCreation);
        }

        StringBuilder cmd = new StringBuilder();
        cmd.append(findExecutable()).append(space)
                .append(src).append(space)
                .append("--footer-right").append(space).append("页码:[page]").append(space)
                .append("--footer-font-size").append(space).append("5").append(space)
                .append("--disable-smart-shrinking").append(space)
                .append("--load-media-error-handling")
                .append(space).append("ignore").append(space)
                .append("--load-error-handling").append(space).append("ignore").append(space)
                .append("--footer-left").append(space).append("电子档打印时间:[date]").append(space)
                .append(dest);

        InputStream is = null;

        try {
            String finalCmd = cmd.toString();
            logger.info("final cmd:{}", finalCmd);

            Process proc = Runtime.getRuntime().exec(finalCmd);
            new Thread(new ProcessStreamHandler(proc.getInputStream())).start();
            new Thread(new ProcessStreamHandler(proc.getErrorStream())).start();
            proc.waitFor();

            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            is = new FileInputStream(file);
            byte[] buf = new byte[1024];
            while (is.read(buf, 0, buf.length) != -1) {
                baos.write(buf, 0, buf.length);
            }

            return baos.toByteArray();
        } catch (Exception e) {
            logger.error("html转换pdf出错", e);
            throw new RuntimeException("html转换pdf出错了");
        } finally {
            if (is != null) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * Attempts to find the `wkhtmltopdf` executable in the system path.
     *
     * @return the wkhtmltopdf command according to the OS
     */
    public String findExecutable() {
        Process p;
        try {
            String osname = System.getProperty("os.name").toLowerCase();

            String cmd = osname.contains("windows") ? "where wkhtmltopdf" : "which wkhtmltopdf";

            p = Runtime.getRuntime().exec(cmd);
            new Thread(new ProcessStreamHandler(p.getErrorStream())).start();
            p.waitFor();

            return IOUtils.toString(p.getInputStream(), Charset.defaultCharset());

        } catch (Exception e) {
            logger.error("no wkhtmltopdf found!", e);
        }

        return "";
    }


    private static class ProcessStreamHandler implements Runnable {

        private InputStream is;

        public ProcessStreamHandler(InputStream is) {
            this.is = is;
        }

        @Override
        public void run() {
            BufferedReader reader = null;

            try {
                InputStreamReader isr = new InputStreamReader(is, "utf-8");
                reader = new BufferedReader(isr);
                String line;
                while ((line = reader.readLine()) != null) {
                    if (ConfigAdapter.getIsDebug())
                        System.out.println(line); //输出内容
                }
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                if (reader != null) {
                    try {
                        reader.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

