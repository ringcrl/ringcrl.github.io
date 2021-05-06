---
layout: post
title: 服务端node导出excel的方案
tags: ['2017']
---

前端浏览器导出 excel 是十分不靠谱的，后台又不配合，怎么办？自己在 node 层撸一个出来咯。


# node-xlsx的使用

node-xlsx文档地址：[https://github.com/mgcrea/node-xlsx](https://github.com/mgcrea/node-xlsx)

node-xlsx依赖于 js-xlsx 组件，API极简单，可读写xlsx文件内容。

node-xlsx的主要使用方法如下：

## 读xlsx文件

```js
let xlsx = require('node-xlsx'),
    fs = require('fs');

const workSheetsFromBuffer = xlsx.parse(fs.readFileSync(`${__dirname}/myFile.xlsx`));
const workSheetsFromFile = xlsx.parse(`${__dirname}/myFile.xlsx`);
```

## 写xlsx文件

```js
let xlsx = require('node-xlsx'),
    fs = require('fs');

const data = [[1, 2, 3], [true, false, null, 'sheetjs'], ['foo', 'bar', new Date('2014-02-19T14:30Z'), '0.3'], ['baz', null, 'qux']];
var buffer = xlsx.build([{name: "mySheetName", data: data}]);

fs.writeFileSync('./test.xlsx', buffer);
```

## 发送excel文件

发送excel文件到浏览器端的关键是设置好 header 的 Content-Type 值。

对于excel，Content-Type有如下两种：

```bash
# .xls 文件
application/vnd.ms-excel

# .xlsx 文件
application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

下面以restify为例写一段demo：

```js
let restify = require('restify'),
    xlsx = require('node-xlsx');

let app = restify.createServer({
    name: 'demo-xlsx',
    version: '1.0.0'
});

// 访问 http://127.0.0.1:8001/test，即下载myfile.xlsx文件
app.get('/test', function(req, res, next) {
    const data = [[1, 2, 3], [true, false, null, 'sheetjs'], ['foo', 'bar', new Date('2014-02-19T14:30Z'), '0.3'], ['baz', null, 'qux']];
    return sendExcel(res, data, 'sheet表名', 'myfile');
});

app.listen(8001, function() {
    console.log(app.name, 'Start listening at %s', app.url);
});

/**
 * 写入Excel
 * @param res            Response对象
 * @param xlsxData       xlsx数组
 * @param sheetName      excel表名
 * @param xlsxFileName   excel文件名（备注：不要使用中文）
 */
function sendExcel(res, xlsxData, sheetName, xlsxFileName) {
    try {
        let buffer = xlsx.build([{name: sheetName, data: xlsxData}]);
        let xlsxContentType = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet';  // For Excel2007 and above .xlsx files

        res.setHeader('Content-Type', xlsxContentType);
        res.setHeader('Content-Disposition', `attachment; filename=${xlsxFileName}.xlsx`);
        res.writeHead(200);
        res.end(buffer);
    } catch (err) {
        console.log('mistake to build excel');
    }
}
```

# nodejs实现导出、下载功能

```js
// 语法糖为coffeeScript
fs = require("fs")
xlsx = require("node-xlsx")
uuid = require("node-uuid")

exports.exportFile = (req,res)->
    data = [[1,2,3],['a','b','c']]
    buffer = xlsx.build([{name:'test',data:data}])
    filePath = __dirname + '/download/' + uuid.v1() + '.xlsx'
    // 创建对应文件
    fs.writeFileSync(filePath,buffer,'binary')
    res.download(filePath)
```

如果只有少量文件可以不用将创建的文件删除（可以作为备份记录），如果文件量大且没有备份的意义，那么就需要在客户端下载完成后将对应的文件删除
