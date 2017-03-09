---
layout:     post
title:      "Android WebView修改加载页面的DOM结构"
subtitle:   "Modify DOM Structure in Android Webview"
date:       2016-12-06 12:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
    - WebView
---


最近工作中要加入一个简单的搜索功能，为用户提供便利。

那有何难？直接用WebView加载一下百度和Google的搜索链接不就完了嘛...
经过分析，百度的搜索url是`https://www.baidu.com/s?wd=keyword`, Google的搜索url是`https://www.google.com/search?q=keyword`。身在天朝，先拿百度做个试验吧~

嗖嗖嗖代码出来了，最简单的WebView使用
```java
private static final String BAIDU_SEARCH_URL = "https://www.baidu.com/s?wd=webtest";
private WebView mWebView;

@Override
protected void onCreate(Bundle savedInstanceState) {    
  super.onCreate(savedInstanceState);    
  mWebView = new WebView(this);    
  setContentView(mWebView);    

  mWebView.loadUrl(BAIDU_SEARCH_URL);
}
```
结果上个图，恩，完美盗用了百度的成果
![加载百度搜索结果](http://upload-images.jianshu.io/upload_images/4075712-949493f4a62e52b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这时候，万恶的pm跳出来了，“咱们能把百度的脑袋砍了吗？”......OK，我懂你兄弟

![砍Baidu的脑袋](http://upload-images.jianshu.io/upload_images/4075712-2b89673c8335c904.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

虽然我也不知道这么做是否侵犯了百度的利益，但是既然pm提出来了，咱们也得给他实现不是。经过一番调查，最后确定了如下方案：
* 加载百度页面
* 加载完成的回调中，向页面注入JavaScript代码
* JavaScript代码将配置的dom元素移除

关键点就是怎么判断页面加载完成以及如何注入砍脑袋的JavaScript代码。

WebView有个回调类叫WebChromeClient，它里面有个`onProgressChanged(WebView view, int newProgress) `的回调方法，每当dom树的加载进度变化时，就通知给我们的app。所以，我们可以近似地认为回调进度是100时就是页面加载完成的时刻。

那怎么确定需要删除的dom元素id呢？我们可以到Chrome的开发者工具找到百度脑袋的dom元素id，如下图所示

![找百度脑袋的id](http://upload-images.jianshu.io/upload_images/4075712-46ae6a9cc47ddb3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好了，万事俱备，上代码
```java
// 需要隐藏的dom元素id
private static final String[] HIDE_DOM_IDS = {"page-hd", "page-tips"};

// 定义WebChromeClient
private WebChromeClient mSearchChromeClient = new WebChromeClient() {  
  @Override    
  public void onProgressChanged(WebView view, int newProgress) {  
    Log.d(SEARCH_TAG, "on page progress changed and progress is " + newProgress);        
    // 进度是100就代表dom树加载完成了
    if (newProgress == 100) {            
      mWebView.loadUrl(getDomOperationStatements(HIDE_DOM_IDS));        
    }    
  }
};

mWebView.setWebChromeClient(mSearchChromeClient);

public static String getDomOperationStatements(String[] hideDomIds) {   
  StringBuilder builder = new StringBuilder();    
  // add javascript prefix    
  builder.append("javascript:(function() { ");   
  for (String domId : hideDomIds) {        
    builder.append("var item = document.getElementById('").append(domId).append("');");        
    builder.append("item.parentNode.removeChild(item);");    
  }    
  // add javascript suffix    
  builder.append("})()");    
  return builder.toString();
}
```

![final.gif](http://upload-images.jianshu.io/upload_images/4075712-90c6af1afdef6e8c.gif?imageMogr2/auto-orient/strip)

可以看到百度脑袋一闪而过，被切掉了。闪这一下倒是比较容易解决，可以通过先`mWebView.setVisibility(View.INVISIBLE)`，执行JavaScript代码1s后再`mWebView.setVisibility(View.VISIBLE)`解决，就不再赘述了。

总结：虽然这个场景实用性不强，但可权且当做WebView操作页面元素的一个例子，提供一种自定义WebView的思路。

