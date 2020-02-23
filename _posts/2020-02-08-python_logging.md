---
layout: post
title:  Python 程序的日志切分
categories: Python
tags: Python
---
* content
{:toc}


Python 脚本或 Web 应用经常使用 logging 模块来输出日志。我们希望日志能优雅地输出，因此要对输出的日志文件进行自动切分，毕竟我们不希望所有的日志都输出在同一个文件中。一般可将日志进行时间切分和大小切分。

目标是：日志文件名中包含日期信息，如 app_20190203.log，并且当文件大小超出限定值后，新建新的文件，如下所示：

```
app_20190203.log
app_20190203.log.1
app_20190203.log.2
app_20190204.log
app_20190205.log
...
```

logging 提供中的 `RotatingFileHandler` 和 `TimedRotatingFileHandler` 用于日志的轮转。对 `RotatingFileHandler` 进行扩展：

```python
class DailyRotatingFileHandler(handlers.RotatingFileHandler):
    def __init__(self, **kwargs):
        self.alias_ = kwargs['filename']
        self.baseFilename = self.getBaseFilename()
        kwargs['filename'] = self.baseFilename
        super().__init__(**kwargs)

    def getBaseFilename(self):
        """
        获取文件名：
        """
        self.today_ = datetime.date.today()
        basename_ = self.alias_ + '_' + self.today_.strftime("%Y%m%d") + '.log'
        return basename_

    def shouldRollover(self, record):
        """
        时间、大小的轮转
        """
        if self.stream is None:
            self.stream = self._open()

        if self.maxBytes > 0 :                  
            msg = "%s\n" % self.format(record)
            self.stream.seek(0, 2)  
            if self.stream.tell() + len(msg) >= self.maxBytes:
                return 1

        if self.today_ != datetime.date.today():
            self.baseFilename = self.getBaseFilename()
            return 1
        return 0
```
