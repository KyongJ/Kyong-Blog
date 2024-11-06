---
sticky: 1000
title: 解决react-csv第三方库出现的解析Bug
author: Kyong
date: 2022-06-21
tag: 
  - Solutions
---

# 解决react-csv第三方库出现的解析Bug

### 一、寻找BUG的原因

最近在公司实习期间接到一个需求，将表格数据导出成CSV格式。在网上查找了一番资料后，决定使用react-csv第三方库来完成需求。

完成需求的过程还是挺顺利的，但是在自测的时候发现当表格字段在双引号内又出现双引号时，csv的解析会出现bug。



> 解析的数据

![image-20230302201544997](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20230302201544997.png)



> csv里的数据

![image-20230302201913819](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20230302201913819.png)

可以发现原本应该在`Last Name`那一列的`Kyong`跑到了`Email`那一列，可见CSV解析的时候出现了bug。

后来查阅了CSV解析规则的时候发现一个规则：

**如果用双引号括字段，那么出现在字段内的双引号前必须加一个双引号进行转义**。

问题出在这了，于是我开始探索源码去解决问题。

### 二、阅读源码

>  Link.jsx

```jsx
class CSVLink extends React.Component {
  static defaultProps = commonDefaultProps;
  static propTypes = commonPropTypes;

  constructor(props) {
    super(props);
    this.buildURI = this.buildURI.bind(this);
  }

  buildURI() {
    return buildURI(...arguments);
  }

  /**
   * In IE11 this method will trigger the file download
   */
  handleLegacy(event, isAsync = false) {
    // If this browser is IE 11, it does not support the `download` attribute
    if (window.navigator.msSaveOrOpenBlob) {
      // Stop the click propagation
      event.preventDefault();

      const {
        data,
        headers,
        separator,
        filename,
        enclosingCharacter,
        uFEFF
      } = this.props;

      const csvData = isAsync && typeof data === 'function' ? data() : data;

      let blob = new Blob([uFEFF ? '\uFEFF' : '', toCSV(csvData, headers, separator, enclosingCharacter)]);
      window.navigator.msSaveBlob(blob, filename);

      return false;
    }
  }

  handleAsyncClick(event) {
    const done = proceed => {
      if (proceed === false) {
        event.preventDefault();
        return;
      }
      this.handleLegacy(event, true);
    };

    this.props.onClick(event, done);
  }

  handleSyncClick(event) {
    const stopEvent = this.props.onClick(event) === false;
    if (stopEvent) {
      event.preventDefault();
      return;
    }
    this.handleLegacy(event);
  }

  handleClick() {
    return event => {
      if (typeof this.props.onClick === 'function') {
        return this.props.asyncOnClick
          ? this.handleAsyncClick(event)
          : this.handleSyncClick(event);
      }
      this.handleLegacy(event);
    };
  }

  render() {
    const {
      data,
      headers,
      separator,
      filename,
      uFEFF,
      children,
      onClick,
      asyncOnClick,
      enclosingCharacter,
      ...rest
    } = this.props;

    const isNodeEnvironment = typeof window === 'undefined';
    const href = isNodeEnvironment ? '' : this.buildURI(data, uFEFF, headers, separator, enclosingCharacter)

    return (
      <a
        download={filename}
        {...rest}
        ref={link => (this.link = link)}
        target="_self"
        href={href}
        onClick={this.handleClick()}
      >
        {children}
      </a>
    );
  }
}

export default CSVLink;
```

删除无用的代码后，可以发现CSVLink的底层是`a`标签，通过`onClick`函数来`download`。而onClick函数调用了`handleLegacy(event, isAsync = false)`这个方法。



> handleLegacy函数

```js
  handleLegacy(event, isAsync = false) {
    // If this browser is IE 11, it does not support the `download` attribute
    if (window.navigator.msSaveOrOpenBlob) {
      // Stop the click propagation
      event.preventDefault();

      const {
        data,
        headers,
        separator,
        filename,
        enclosingCharacter,
        uFEFF
      } = this.props;

      const csvData = isAsync && typeof data === 'function' ? data() : data;

      let blob = new Blob([uFEFF ? '\uFEFF' : '', toCSV(csvData, headers, separator, enclosingCharacter)]);
      window.navigator.msSaveBlob(blob, filename);

      return false;
    }
  }
```

`handleLegacy`函数的主要作用就是得到用户传进来的数据，然后使用`toCSV`函数将用户的数据转换为`CSV`能够解析的格式，然后将文件下载下来。接下来我们看`toCSV`函数。



> toCSV

```js
export const toCSV = (data, headers, separator, enclosingCharacter) => {
  if (isJsons(data)) return jsons2csv(data, headers, separator, enclosingCharacter);
  if (isArrays(data)) return arrays2csv(data, headers, separator, enclosingCharacter);
  if (typeof data === 'string') return string2csv(data, headers, separator);
  throw new TypeError(`Data should be a "String", "Array of arrays" OR "Array of objects" `);
};

export const arrays2csv = ((data, headers, separator, enclosingCharacter) =>
  joiner(headers ? [headers, ...data] : data, separator, enclosingCharacter)
);

export const joiner = ((data, separator = ',', enclosingCharacter = '"') => {
  let res =  data
    .filter(e => e)
    .map(
      row => row
        .map((element) => elementOrEmpty(element))
        .map(column => `${enclosingCharacter}${column}${enclosingCharacter}`)
        .join(separator)
    )
    .join(`\n`);
  return res;
});
```

`toCSV`判断传进来的数据的格式，从而确定去调用哪一个方法。因为我们测试的时候传进来的是一个数组，所以重点看`arrays2csv`函数。`arrays2csv`函数底层调用了`joiner`函数，而`joiner`函数正式数组转为csv解析格式的核心。

`joiner`函数通过`map`函数`column`转换为`${enclosingCharacter}${column}${enclosingCharacter}`，而问题正是出在这，我们可以将其代码稍微修改一下，通过字符串的`replace`函数将一个双引号替代为两个双引号从而解决问题。

### 三、修改源码

原引用公共库 [react-csv](https://www.npmjs.com/package/react-csv)。但有个小 bug ：当单元格内含有双引号 `"` 时，会与 csv 的分隔符冲突，暂时还没有提交 pr ，故自行修改，等 pr 通过之后再换回。

修改 `core.js` 下的 `joiner` 函数：	

```js
export const joiner = (data, separator = ',', enclosingCharacter = '"') => {
  return data
    .filter((e) => e)
    .map((row) =>
      row
        .map((element) => elementOrEmpty(element))
        .map((column) => `${enclosingCharacter}${column}${enclosingCharacter}`)
        .join(separator),
    )
    .join(`\n`);
};
```

改为：

```js
export const joiner = (data, separator = ',', enclosingCharacter = '"') => {
  return data
    .filter((e) => e)
    .map((row) =>
      row
        .map((element) => elementOrEmpty(element))
        .map((column) => `${enclosingCharacter}${String(column).replace(/"/g, '""')}${enclosingCharacter}`)
        .join(separator),
    )
    .join(`\n`);
};
```