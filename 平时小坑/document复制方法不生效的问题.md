
前端复制不能用是一个很常见的问题，还是写下来凑一篇文档数～

## 1. 在非安全域网址中不生效

常用navigator.clipboard.writeText(data);但是这个在非安全域网址中不能用

```js
function handleCopy() {
	const reportText = getReportContent(groupData.value);
	if (navigator.clipboard) {
		navigator.clipboard
		.writeText(reportText)
		.then(() => {
			message.success('已复制到剪切板！');
		})
		.catch((err) => {
			console.error('复制失败: ', err);
		});
	} else {
		console.warn('当前为非安全域，已使用其他复制指令');
		// 创建text area
		const textArea = document.createElement('textarea');
		textArea.style.height = '0';
		document.body.appendChild(textArea);
		textArea.value = reportText;
		textArea.focus();
		textArea.select();
		return new Promise((res, rej) => {
			//  传统方法，兼容性好但已废弃
			document.execCommand('copy') ? res('success') : rej(); 
			message.success('已复制到剪切板！');
			textArea.remove();
		});
	}
}
```

## 2. 在Model等脱离文档流的地方不生效
```js
try {
	if (navigator.clipboard) {
		await navigator.clipboard.writeText(data);
	} else {
		throw new Error('');
	}
} catch (error) {
	const textarea = document.createElement("textarea")
	 // 弹出框(脱离文档流)的Element元素
	const container = document.querySelector("#apiKeyModel")
	container?.appendChild(textarea)
	textarea.value = data
	textarea.select()
	document.execCommand("copy")
	container?.removeChild(textarea)
}
```