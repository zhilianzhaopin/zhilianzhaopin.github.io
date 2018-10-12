---
layout: post
title: "智联路由ZPRoute（iOS）"
subtitle: "iOS Router"
author: "Ren Lei"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.4
tags:
  - iOS
  - 组件化
  - 路由
  - Router
---

## iOS ZPRoute
```
ZPRoute是根据Android ARouter设计思想实现的
```

#### 一、功能介绍
1. 支持解析标准URL进行跳转，支持传递参数
2. 支持回调传值
3. 支持用户指定全局降级与局部降级策略
4. 支持添加多个全局拦截器和某个页面自定义拦截
5. 支持动态修改路径
6. 支持模块解耦（供应者）
7. 支持第三方APP跳转

#### 二、典型应用
1. 从外部URL映射到内部页面，以及参数传递与解析
2. 跨模块页面跳转，模块间解耦
3. 拦截跳转过程，处理登陆、埋点等逻辑
4. 跨模块API调用，通过控制反转来做组件解耦


#### 三、基础功能
1.配置Plist路径 如下示例
``` 
{
zhaopin =     {
ARouteModule =         {
native =             {
test =                 {
class = ARouteViewController;
};
};
};
Resume =         {
native =             {
Resume1 =                 {
class = ResumeViewController;
};
};
};
webModule =         {
h5 =             {
page1 =                 {
class = WebViewController;
xib = WebViewController;
};
page2 =                 {
class = WebViewController;
xib = WebViewController;
interception =                     {
class = Page2Interception;
};
};
};
};
};
}
```
对应URL路径格式为
```
scheme://host/module/native/className?key=value
scheme://host/module/h5/className?key=value
scheme://host/module/weex/className?key=value
```
如 zhaopin://zhaopin.com/webModule/h5/page2  
页面属性配置包含:   
`class` 页面名称  
`xib`  如果是xib初始化则必填属性  
`interception` 页面自定义拦截器  

2.初始化
```
ZPRouteConfig *config = [ZPRoute route].config;  
config.nativeScheme = @"zhaopin"; // 与plist对应scheme  
config.nativeHost = @"zhaopin.com"; // 暂未使用此属性  

// 注册plist  
[config addRouteWithPlistPaths:[self routePlist]];  

// 设置解析规则
[config setRouteParserRuleBlock:^NSArray *(NSString *path) {
NSMutableArray *array = [NSMutableArray arrayWithObject:@"zhaopin"];
[array addObjectsFromArray: ({
[([path hasPrefix:@"/"] ? [path substringFromIndex:1] : path) pathComponents];
})];
return array;
}];
```
3.Pod引入

```
source 'http://gitlab.dev.zhaopin.com/lei.ren/ZPRoute.git'

pod 'ZPRoute',:git => 'http://gitlab.dev.zhaopin.com/lei.ren/ZPRoute.git'

#import <ZPRoute/UIViewController+ZPRoute.h>

```

4.发起路由
```
// 简单跳转
url = @"/webModule/h5/page1";
self.route.build(url,self).withGreenChannal().navigation(nil);

// 简单带参跳转
url = @"/webModule/h5/page1";

NSDictionary *dic = @{@"localUrl":@"routeHTML.html"};

self.route.build(url,self).withParameter(dic).withGreenChannal().navigation(nil);

// 回调处理
url = @"/webModule/h5/page1";

NSDictionary *dic = @{@"localUrl":@"routeHTML.html"};

self.route.build(url,self).withParameter(dic).withGreenChannal().navigation(^(ZPRouteCallback *callback) {

callback.onCallback = ^(NSDictionary *dic) {
NSLog(@"%@回调处理 == %@",self,dic);
};
});

// 找不到页面，自定义降级
url = @"/webModule/h5/page1111111";

self.route.build(url,self).withGreenChannal().navigation(^(ZPRouteCallback *callback) {

callback.onLost = ^{
UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"找不到URL" message:@"自定义降级" preferredStyle:UIAlertControllerStyleAlert];
UIAlertAction *ok = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:nil];
[alert addAction:ok];
[self presentViewController:alert animated:YES completion:nil];
};
});

// 打开外部APP 如safai
url = @"http://www.baidu.com";
self.route.build(url, nil).withOpenMode(openExternal).navigation(nil);

```

5.自定义全局降级策略

```

// 实现ZPRouteDegradeProtocol接口，注：全局降级只有1个

@interface Degrade ()<ZPRouteDegradeProtocol>

@end

@implementation Degrade


- (void)onLost:(ZPRoutePostcard *)postcard {
NSLog(@"全局降级 parame = %@",postcard.originalURL);

DegradeViewController *vc = [[DegradeViewController alloc] init];
vc.url = postcard.path;
[postcard.lastViewController.navigationController pushViewController:vc animated:YES];
}

@end

```
6.自定义拦截

```

// 实现ZPRouteInterceptorProtocol接口

@interface Page2Interception ()<ZPRouteInterceptorProtocol>

@end

@implementation Page2Interception

- (void)interceptorProcess:(ZPRoutePostcard *)postcard complete:(void (^)(BOOL))block {

UIAlertController *alert = [UIAlertController alertControllerWithTitle:postcard.path message:@"自定义拦截" preferredStyle:UIAlertControllerStyleAlert];
UIAlertAction *ok = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
block(YES);
}];
[alert addAction:ok];

UIAlertAction *cancel = [UIAlertAction actionWithTitle:@"cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
block(NO);
}];
[alert addAction:cancel];

[[UIApplication sharedApplication].keyWindow.rootViewController presentViewController:alert animated:YES completion:nil];
}

@end

```


7.全局拦截需实现ZPRouteGlobalInterceptorProtocol，全局拦截可注册多个，多个拦截器会按优先级顺序依次执行
```

@interface LoginInterceptor ()<ZPRouteGlobalInterceptorProtocol>

@end

@implementation LoginInterceptor

//  优先级
- (NSInteger)priority {
return 3;
}

- (void)interceptorProcess:(ZPRoutePostcard *)postcard complete:(void (^)(BOOL))block {

NSLog(@"登录拦截器");

//    if(是否登录判断) {
//
//    }

// 登录逻辑
LoginViewController *vc = [[LoginViewController alloc] init];

vc.loginResultBlock = ^(BOOL isLogin) {

NSLog(@"登录 --> %@",isLogin ? @"成功" : @"失败");

block(isLogin);
};

UIViewController *lastViewController = postcard.lastViewController;

UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:vc];
[lastViewController presentViewController:nav animated:YES completion:nil];

}




@end
```


8.通过依赖注入解耦:服务管理(一) 暴露服务
```
// 声明接口,其他组件通过接口来调用服务 接口必须继承于ZPRouteProviderProtocol用于判断合法性

@protocol DataProvider <ZPRouteProviderProtocol>

- (NSString *)name;

- (void)asyncName:(void(^)(NSString *name))block;

@end

// 实现接口

#import "DataProvider.h"

@interface Data ()<DataProvider>

@end

@implementation Data

/// 必须实现此方法，返回实例对象
+ (id)routeProviderInstance {
 return [[self alloc] init];
}

- (NSString *)name {

return @"ZPRoute hello";
}

- (void)asyncName:(void(^)(NSString *name))block {

dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
block(@"ZPRoute hellooooooo");
});
}

@end

```


9.通过依赖注入解耦:服务管理(二) 发现服务

```

// 同步获取
id <DataProvider> data = [self.route getProvider:@protocol(DataProvider)];

NSLog(@"同步获取 = %@",data.name);

// 异步获取

id <DataProvider> data = [self.route getProvider:@protocol(DataProvider)];

NSLog(@"异步获取开始 3秒后回调");

[data asyncName:^(NSString *name) {

NSLog(@"异步获取 = %@",name);
}];

```

10.回调传递参数到上级页面
```
// 在二级页面向一级页面回调
[self routeCallBack:@{@"111":@"2222"}];

```

11.获取上级页面传递参数
```
NSString *url = self.postcard.parameter[@"localUrl"];
```

12.动态修改路径
```
NSString *oldPath = @"/webModule/h5/page2";
NSString *newPath = @"/Resume/native/Resume1";
[self.route.config updateRouteOldPath:oldPath newPath:newPath];
```

13.根据URL获取某个页面信息
```
url = @"/webModule/h5/page2";

NSDictionary *dict = self.route.fetchPageInfo(url);
```

## 注意
URL支持以下写法
```
scheme://host/module/type/name
//host/module/type/name
/module/type/name
```

**Config**必须设置解析规则，否则ZPRoute无法正常工作

全局拦截，全局降级，供应者服务，只需实现对应接口即可，ZPRouteConfig会自动将其进行注册





