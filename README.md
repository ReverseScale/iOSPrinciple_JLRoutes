# iOSPrinciple_JLRoutes
Principle JLRoutes

### 内部流程：
#### 1）注册流程
* 调用 addRoute:handler: 方法注册 url scheme，保存/取出每个 scheme 对应的 routes controller 对象（以 key-value 形式保存的）；
* 注册 url pattern，按优先级将每个 pattern 对应的 JLRRouteDefinition（封装 pattern、priority、回调 block 等数据）通过插入排序的方式保存到数组中，这里一个 pattern 可能对应一个或者多个 JLRRouteDefinition。

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/60187702.jpg)

#### 2）解析 URL 流程
调用 routeURL: 方法唤起 URL时，取出 scheme 对应的 routes controller，生成 JLRRouteRequest，然后遍历所有注册过的 JLRRouteDefinition，遍历时每个 JLRRouteDefinition 会根据 request 进行匹配并生成一个  JLRRouteResponse 对象，如果匹配上了，就回调 block，并回传参数。

### 基本用法

#### 1）配置 URL Schemes
一个 app 可以对应多个 URL Schemes，如下图 info.plist 配置，在 Safari 中，只要输入 JLRouteSchemeOne:// 或 JLRouteSchemeTwo:// 都可以打开该 app，而 URL identifier 最好是保证其唯一性。

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/73284895.jpg)

#### 2）注册 JLRoutes

注册路由的方式有很多种：

① 全局JLRoutes注册

```objc
[[JLRoutes globalRoutes] addRoute:@"取url内容值的标识" handler:^BOOL(NSDictionary<NSString *,id> * _Nonnull parameters) {
    return YES; // 一旦匹配，立即返回 YES
}];
```

② 自定义命名空间注册

```objc
[[JLRoutes routesForScheme:@"第一模块的标识"] addRoute:@"取url内容值的标识" handler:^BOOL(NSDictionary<NSString *,id> * _Nonnull parameters) {
    return YES; // 一旦匹配，立即返回 YES
}];
```

③ 定义优先级注册

```objc
[[JLRoutes globalRoutes] addRoute:@"取url内容值的标识" priority:1 handler:^BOOL(NSDictionary<NSString *,id> * _Nonnull parameters) {
    return YES; // 一旦匹配，立即返回 YES
}];
```
如果不设置优先级，所有的注册优先级都为 0。

当标识了优先级进行注册后，JLRRouteDefinition 对象(最终模型)在 JLRoutes 对象的 routes 数组 中将进行排序，类似于选择排序，当通过route对象寻找到其 routes 数组 后，将会遍历整个 routes 数组，优先级高的 JLRRouteDefinition 对象将会被最先匹配，然后 return YES，并停止遍历。


④ 定义多个 "取url内容值的标识" 进行注册

```objc
[[JLRoutes globalRoutes] addRoutes:@[@"取url内容值的标识", @"取url内容值的标识"] handler:^BOOL(NSDictionary<NSString *,id> * _Nonnull parameters) {
    return YES; // 一旦匹配，立即返回 YES
}];
```

#### 3）点击跳转

将 URL scheme 添加进了 info.plist，并进行注册，接下来使用如下方法进行跳转：

```objc
NSString *customURL = @"JLRouteSchemeOne://OneDetailViewController";
    [[UIApplication sharedApplication] openURL:[NSURL URLWithString:customURL]];
```

#### 4）参数传递
参数传递需要进行一一对应

```objc
[[JLRoutes routesForScheme:@"JLRouteSchemeTwo"]addRoute:@"/:ViewController/:userID/:pass" handler:^BOOL(NSDictionary<NSString *,id> * _Nonnull parameters) {
    
    NSLog(@"parameters: %@",parameters);
    NSLog(@"userID: %@",parameters[@"userID"]);
    NSLog(@"pass: %@",parameters[@"pass"]);

    Class class = NSClassFromString(parameters[@"ViewController"]);
    [navVc pushViewController:[[class alloc]init] animated:YES];
    
    return YES;
}];
```

点击方法如下：

```objc
- (void)clickBtn {
    NSString *customURL = @"JLRouteSchemeTwo://TwoDetailViewController/我是userID/我是pwd";
    // 中文传输需要进行转义
    customURL = [customURL stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet URLQueryAllowedCharacterSet]];
    [[UIApplication sharedApplication] openURL:[NSURL URLWithString:customURL]];
}
```

处理从网页等跳转过来，比如直接跳到第二模块的第二级控制器

#### 5）实现 openURL 方法

openURL方法在此处对所有的跳转进行拦截，手动解析处理，再交于JLRoutes。

```objc
- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options {
    // 默认的路由 跳转等等
    if ([[url scheme] isEqualToString:DefaultRouteSchema]) {
        return [[JLRoutes globalRoutes] routeURL:url];
    }
    // http
    else if ([[url scheme] isEqualToString:HTTPRouteSchema]) {
        return [[JLRoutes routesForScheme:HTTPRouteSchema] routeURL:url];
    }
    // https
    else if ([[url scheme] isEqualToString:HTTPsRouteSchema]) {
        return [[JLRoutes routesForScheme:HTTPsRouteSchema] routeURL:url];
    }
    // Web交互请求
    else if ([[url scheme] isEqualToString:WebHandlerRouteSchema]) {
        return [[JLRoutes routesForScheme:WebHandlerRouteSchema] routeURL:url];
    }
    // 请求回调
    else if ([[url scheme] isEqualToString:ComponentsCallBackHandlerRouteSchema]) {
        return [[JLRoutes routesForScheme:ComponentsCallBackHandlerRouteSchema] routeURL:url];
    }
    // 未知请求
    else if ([[url scheme] isEqualToString:UnknownHandlerRouteSchema]) {
        return [[JLRoutes routesForScheme:UnknownHandlerRouteSchema] routeURL:url];
    }
    return NO;
}
```

### 源码分析
#### 1）框架结构
* JLRoutes
作为 JLRoutes 框架的入口，负责注册 URL，管理路由以及分配路由。
* JLRRouteDefinition
用来封装注册 URL 的路由信息，包括 URL scheme, route pattern, and priority，并且可以根据 request 提供相应的 response。可以通过继承该类来实现自定义的匹配方式。
* JLRRouteRequest
用来封装一个 URL 的路由请求信息，包括 URL、解析后的 path components 和 query parameters。
* JLRRouteResponse
根据 URL 匹配路由信息时的 response，包含 isMatch、parameters 等信息。如果 JLRRouteDefinition 匹配 URL 成功时，就会设置属性 isMatch 为 YES，同时将解析 URL 后的参数和 默认参数、附加参数组合返回。
* JLRRouteHandler 和 JLRRouteHandlerTarget
自定义路由 handler，也就是将回调参数处理的逻辑交给自定义类去处理。
* JLRParsingUtilities
解析 URL 参数的工具类。

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/18488165.jpg)

halfrost的博客整理的 JLRoutes 数据存储的数据结构

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/97802579.jpg)

JLRoutes会传入每个字符串，都按照上面的样子进行切分处理，分别根据RFC的标准定义，取到各个NSURLComponent。

JLRoutes全局会保存一个Map，这个Map会以scheme为Key，JLRoutes为Value。所以在routeControllerMap里面每个scheme都是唯一的。

在每个JLRoutes里面都保存了一个数组，这个数组里面保存了每个路由规则JLRRouteDefinition里面会保存外部传进来的block闭包，pattern，和拆分之后的pattern。

在每个JLRoutes的数组里面，会按照路由的优先级进行排列，优先级高的排列在前面。

```objc
- (void)_registerRoute:(JLRRouteDefinition *)route
{
    if (route.priority == 0 || self.mutableRoutes.count == 0) {
        [self.mutableRoutes addObject:route];
    } else {
        NSUInteger index = 0;
        BOOL addedRoute = NO;
        
        // search through existing routes looking for a lower priority route than this one
        for (JLRRouteDefinition *existingRoute in [self.mutableRoutes copy]) {
            if (existingRoute.priority < route.priority) {
                // if found, add the route after it
                [self.mutableRoutes insertObject:route atIndex:index];
                addedRoute = YES;
                break;
            }
            index++;
        }
        // if we weren't able to find a lower priority route, this is the new lowest priority route (or same priority as self.routes.lastObject) and should just be added
        if (!addedRoute) {
            [self.mutableRoutes addObject:route];
        }
    }
}
```

由于这个数组里面的路由是一个单调队列，所以查找优先级的时候只用从高往低遍历即可。

具体查找路由的过程如下：

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/32310864.jpg)

首先根据外部传进来的URL初始化一个JLRRouteRequest，然后用这个JLRRouteRequest在当前的路由数组里面依次request，每个规则都会生成一个response，但是只有符合条件的response才会match，最后取出匹配的JLRRouteResponse拿出其字典parameters里面对应的参数就可以了。

查找和匹配过程中重要的代码如下：

```objc
- (BOOL)_routeURL:(NSURL *)URL withParameters:(NSDictionary *)parameters executeRouteBlock:(BOOL)executeRouteBlock
{
    if (!URL) {
        return NO;
    }
    [self _verboseLog:@"Trying to route URL %@", URL];
    
    BOOL didRoute = NO;
    JLRRouteRequest *request = [[JLRRouteRequest alloc] initWithURL:URL alwaysTreatsHostAsPathComponent:alwaysTreatsHostAsPathComponent];
    
    for (JLRRouteDefinition *route in [self.mutableRoutes copy]) {
        // check each route for a matching response
        JLRRouteResponse *response = [route routeResponseForRequest:request decodePlusSymbols:shouldDecodePlusSymbols];
        if (!response.isMatch) {
            continue;
        }
        [self _verboseLog:@"Successfully matched %@", route];
        
        if (!executeRouteBlock) {
            // if we shouldn't execute but it was a match, we're done now
            return YES;
        }
        // configure the final parameters
        NSMutableDictionary *finalParameters = [NSMutableDictionary dictionary];
        [finalParameters addEntriesFromDictionary:response.parameters];
        [finalParameters addEntriesFromDictionary:parameters];
        [self _verboseLog:@"Final parameters are %@", finalParameters];
        
        didRoute = [route callHandlerBlockWithParameters:finalParameters];
        if (didRoute) {
            // if it was routed successfully, we're done
            break;
        }
    }
    if (!didRoute) {
        [self _verboseLog:@"Could not find a matching route"];
    }
    // if we couldn't find a match and this routes controller specifies to fallback and its also not the global routes controller, then...
    if (!didRoute && self.shouldFallbackToGlobalRoutes && ![self _isGlobalRoutesController]) {
        [self _verboseLog:@"Falling back to global routes..."];
        didRoute = [[JLRoutes globalRoutes] _routeURL:URL withParameters:parameters executeRouteBlock:executeRouteBlock];
    }
    // if, after everything, we did not route anything and we have an unmatched URL handler, then call it
    if (!didRoute && executeRouteBlock && self.unmatchedURLHandler) {
        [self _verboseLog:@"Falling back to the unmatched URL handler"];
        self.unmatchedURLHandler(self, URL, parameters);
    }
    return didRoute;
}
```

*举个🌰*

先注册一个Router，规则如下：

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/99020849.jpg)

我们传入一个URL，让Router进行处理。

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/66735347.jpg)

匹配成功之后，我们会得到下面这样一个字典：

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/27458383.jpg)

把上述过程图解出来，见下图：

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/70566617.jpg)

> JLRoutes还可以支持Optional的路由规则

假如定义一条路由规则：

```c 
/the(/foo/:a)(/bar/:b)
``` 

JLRoutes 会帮我们默认注册如下4条路由规则：

```c
/the/foo/:a/bar/:b
/the/foo/:a
/the/foo/:b
/the
```

#### 2）核心代码

2.1 routesForScheme函数

routesForScheme函数根据scheme找对应的routes。

```objc
/// Returns a routing namespace for the given scheme
+ (instancetype)routesForScheme:(NSString *)scheme {
    JLRoutes *routesController = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        routeControllersMap = [[NSMutableDictionary alloc] init];
    });
    if (!routeControllersMap[scheme]) {
        routesController = [[self alloc] init];
        routesController.scheme = scheme;
        routeControllersMap[scheme] = routesController;
    }
    routesController = routeControllersMap[scheme];
    return routesController;
}
```

routeControllersMap是一个字典，key是scheme，也就是命名空间；value是一个JLRoutes实例，这个JLRoutes有个成员变量叫mutableRoutes，存放所有的这个命名空间下的routes。

2.2 addRoute函数

addRoute 函数分四部分，第一部分是处理可变的参数，第二部分是创建一个自定义的实例JLRRouteDefinition，第三部分处理一下可变参数的路由注册问题(遍历所有可能注册)，第四部分注册路由。

```objc
- (void)addRoute:(NSString *)routePattern priority:(NSUInteger)priority handler:(BOOL (^)(NSDictionary<NSString *, id> *parameters))handlerBlock {
    //#01 处理可变的参数
    NSArray <NSString *> *optionalRoutePatterns = [JLRParsingUtilities expandOptionalRoutePatternsForPattern:routePattern];
    //#02 创建一个自定义的实例JLRRouteDefinition
    JLRRouteDefinition *route = [[JLRRouteDefinition alloc] initWithScheme:self.scheme pattern:routePattern priority:priority handlerBlock:handlerBlock];
    //#03 遍历可变参数的路由注册
    if (optionalRoutePatterns.count > 0) {
        // there are optional params, parse and add them
        for (NSString *pattern in optionalRoutePatterns) {
            [self _verboseLog:@"Automatically created optional route: %@", route];
            JLRRouteDefinition *optionalRoute = [[JLRRouteDefinition alloc] initWithScheme:self.scheme pattern:pattern priority:priority handlerBlock:handlerBlock];
            [self _registerRoute:optionalRoute];
        }
        return;
    }
    //#04 注册路由
    [self _registerRoute:route];
}
```

分步讲解一下

2.2.1 处理可变的参数

如果来了一种带括号的url：/path/:thing/(/a)(/b)(/c)，我们把它拆分组合成如下一个array。

```objc
/path/:thing/a/b/c
/path/:thing/a/b
/path/:thing/a/c
/path/:thing/b/a
/path/:thing/a
/path/:thing/b
/path/:thing/c
```

```objc
NSArray <NSString *> *optionalRoutePatterns = [JLRParsingUtilities expandOptionalRoutePatternsForPattern:routePattern];
```

expandOptionalRoutePatternsForPattern 实现方法

```objc
+ (NSArray <NSString *> *)expandOptionalRoutePatternsForPattern:(NSString *)routePattern {
    
    if ([routePattern rangeOfString:@"("].location == NSNotFound) {
        return @[];
    }
    
    NSString *baseRoute = nil;
    NSArray *components = [self _optionalComponentsForPattern:routePattern baseRoute:&baseRoute];
    NSArray *routes = [self _routesForOptionalComponents:components baseRoute:baseRoute];
    
    return routes;
}
```

判断保护不必多说，_optionalComponentsForPattern方法中会用到while循环scanUpToString，截取到指定字符串，把参数都截取下来。

```objc
    while ([scanner scanUpToString:@"(" intoString:&nonOptionalRouteSubpath]) {}
```

_routesForOptionalComponents主要用于合并组合返回字符串的数组routes。

```objc
    if (optionalComponents.count == 0 || baseRoute.length == 0) {
        return @[];
    }
    NSMutableArray *routes = [NSMutableArray array];
    // generate all possible combinations of the components that could exist (taking order into account)
    // aka, "/path/:thing/(/a)(/b)(/c)" should never generate a route for "/path/:thing/(/b)(/a)"
    NSArray *combinations = [optionalComponents JLRoutes_allOrderedCombinations];
    // generate the actual final route path strings
    for (NSArray *components in combinations) {
        NSString *path = [components componentsJoinedByString:@""];
        [routes addObject:[baseRoute stringByAppendingString:path]];
    }
    // sort them so that the longest routes are first (since they are the most selective)
    [routes sortUsingSelector:@selector(length)];
    return [routes copy];
```

2.2.2 创建一个自定义的实例JLRRouteDefinition

初始化了一个用来存放route的实例，初始化方法如下

```objc
- (instancetype)initWithScheme:(NSString *)scheme pattern:(NSString *)pattern priority:(NSUInteger)priority handlerBlock:(BOOL (^)(NSDictionary *parameters))handlerBlock {
    if ((self = [super init])) {
        self.scheme = scheme;
        self.pattern = pattern;
        self.priority = priority;
        self.handlerBlock = handlerBlock;
        if ([pattern characterAtIndex:0] == '/') {
            pattern = [pattern substringFromIndex:1];
        }
        self.patternComponents = [pattern componentsSeparatedByString:@"/"];
    }
    return self;
}
```

实例里有scheme命名空间，pattern模式，priority优先级，和要执行的handlerBlock。

2.2.3 遍历可变参数的路由注册

```objc
if (optionalRoutePatterns.count > 0) {
        // there are optional params, parse and add them
        for (NSString *pattern in optionalRoutePatterns) {
            [self _verboseLog:@"Automatically created optional route: %@", route];
            JLRRouteDefinition *optionalRoute = [[JLRRouteDefinition alloc] initWithScheme:self.scheme pattern:pattern priority:priority handlerBlock:handlerBlock];
            [self _registerRoute:optionalRoute];
        }
        return;
    }
```

遍历optionalRoutePatterns创建注册路由对象

2.2.4 注册路由

```objc
- (void)_registerRoute:(JLRRouteDefinition *)route {
    if (route.priority == 0 || self.mutableRoutes.count == 0) {
        [self.mutableRoutes addObject:route];
    } else {
        NSUInteger index = 0;
        BOOL addedRoute = NO;
        
        // search through existing routes looking for a lower priority route than this one
        for (JLRRouteDefinition *existingRoute in [self.mutableRoutes copy]) {
            if (existingRoute.priority < route.priority) {
                // if found, add the route after it
                [self.mutableRoutes insertObject:route atIndex:index];
                addedRoute = YES;
                break;
            }
            index++;
        }
        // if we weren't able to find a lower priority route, this is the new lowest priority route (or same priority as self.routes.lastObject) and should just be added
        if (!addedRoute) {
            [self.mutableRoutes addObject:route];
        }
    }
}
```

在某个scheme命名空间下，如果从没有注册过，addObject:方法添加，如果有路由存在，遍历并根据优先级插入。

2.3 匹配函数

匹配函数包括创建请求URL的JLRRouteRequest对象、遍历匹配URL、response的参数处理和globalRoutes匹配。

```objc
- (BOOL)_routeURL:(NSURL *)URL withParameters:(NSDictionary *)parameters executeRouteBlock:(BOOL)executeRouteBlock {
    ...
    
    BOOL didRoute = NO;
    #1
    JLRRouteRequest *request = [[JLRRouteRequest alloc] initWithURL:URL alwaysTreatsHostAsPathComponent:alwaysTreatsHostAsPathComponent];
    
    #2
    for (JLRRouteDefinition *route in [self.mutableRoutes copy]) {
        
        JLRRouteResponse *response = [route routeResponseForRequest:request decodePlusSymbols:shouldDecodePlusSymbols];
        if (!response.isMatch) {
            continue;
        }
        
        [self _verboseLog:@"Successfully matched %@", route];
        
        if (!executeRouteBlock) {
            return YES;
        }
        
        #3
        NSMutableDictionary *finalParameters = [NSMutableDictionary dictionary];
        [finalParameters addEntriesFromDictionary:response.parameters];
        [finalParameters addEntriesFromDictionary:parameters];
        [self _verboseLog:@"Final parameters are %@", finalParameters];
        
        didRoute = [route callHandlerBlockWithParameters:finalParameters];
        
        if (didRoute) {
            break;
        }
    }
    
    if (!didRoute) {
        [self _verboseLog:@"Could not find a matching route"];
    }
    
    #4
    if (!didRoute && self.shouldFallbackToGlobalRoutes && ![self _isGlobalRoutesController]) {
        [self _verboseLog:@"Falling back to global routes..."];
        didRoute = [[JLRoutes globalRoutes] _routeURL:URL withParameters:parameters executeRouteBlock:executeRouteBlock];
    }
    
    if (!didRoute && executeRouteBlock && self.unmatchedURLHandler) {
        [self _verboseLog:@"Falling back to the unmatched URL handler"];
        self.unmatchedURLHandler(self, URL, parameters);
    }
    
    return didRoute;
}
```

2.3.1 创建JLRRouteRequest对象

创建一个JLRRouteRequest对象，对象包含三个参数：URL，pathComponents，queryParams。

initWithURL:函数其实就是根据传入的URL解析成上面几个参数。

```objc
/**
 Creates a new route request.
 
 @param URL The URL to route.
 @param alwaysTreatsHostAsPathComponent The global option for if to treat the URL host as a path component or not.
 
 @returns The newly initialized route request.
 */
- (instancetype)initWithURL:(NSURL *)URL alwaysTreatsHostAsPathComponent:(BOOL)alwaysTreatsHostAsPathComponent NS_DESIGNATED_INITIALIZER;
```

用张图片介绍，方便理解init函数

![](http://og1yl0w9z.bkt.clouddn.com/18-5-29/60081111.jpg)

2.3.2 遍历匹配URL

解决匹配和存储两个问题。

遍历每一个self.mutableRoutes，取出URLPattern，和刚刚生成的JLRRouteRequest进行匹配，匹配结果封装在JLRRouteResponse中。

```objc
- (JLRRouteResponse *)routeResponseForRequest:(JLRRouteRequest *)request decodePlusSymbols:(BOOL)decodePlusSymbols
{
    BOOL patternContainsWildcard = [self.patternComponents containsObject:@"*"];
    
    if (request.pathComponents.count != self.patternComponents.count && !patternContainsWildcard) {
        // definitely not a match, nothing left to do
        return [JLRRouteResponse invalidMatchResponse];
    }
    
    JLRRouteResponse *response = [JLRRouteResponse invalidMatchResponse];
    NSMutableDictionary *routeParams = [NSMutableDictionary dictionary];
    BOOL isMatch = YES;
    NSUInteger index = 0;
    
    for (NSString *patternComponent in self.patternComponents) {
        NSString *URLComponent = nil;
        
        // figure out which URLComponent it is, taking wildcards into account
        if (index < [request.pathComponents count]) {
            URLComponent = request.pathComponents[index];
        } else if ([patternComponent isEqualToString:@"*"]) {
            // match /foo by /foo/*
            URLComponent = [request.pathComponents lastObject];
        }
        
        if ([patternComponent hasPrefix:@":"]) {
            // this is a variable, set it in the params
            NSString *variableName = [self variableNameForValue:patternComponent];
            NSString *variableValue = [self variableValueForValue:URLComponent decodePlusSymbols:decodePlusSymbols];
            routeParams[variableName] = variableValue;
        } else if ([patternComponent isEqualToString:@"*"]) {
            // match wildcards
            NSUInteger minRequiredParams = index;
            if (request.pathComponents.count >= minRequiredParams) {
                // match: /a/b/c/* has to be matched by at least /a/b/c
                routeParams[JLRouteWildcardComponentsKey] = [request.pathComponents subarrayWithRange:NSMakeRange(index, request.pathComponents.count - index)];
                isMatch = YES;
            } else {
                // not a match: /a/b/c/* cannot be matched by URL /a/b/
                isMatch = NO;
            }
            break;
        } else if (![patternComponent isEqualToString:URLComponent]) {
            // break if this is a static component and it isn't a match
            isMatch = NO;
            break;
        }
        index++;
    }
    
    if (isMatch) {
        // if it's a match, set up the param dictionary and create a valid match response
        NSMutableDictionary *params = [NSMutableDictionary dictionary];
        [params addEntriesFromDictionary:[JLRParsingUtilities queryParams:request.queryParams decodePlusSymbols:decodePlusSymbols]];
        [params addEntriesFromDictionary:routeParams];
        [params addEntriesFromDictionary:[self baseMatchParametersForRequest:request]];
        response = [JLRRouteResponse validMatchResponseWithParameters:[params copy]];
    }
    return response;
}
```

匹配解析：

for循环遍历self.patternComponents，同时取URLComponent，针对每次取出来的一部分进行匹配：

* 包括是否有:作为前缀的匹配
* 是否有*作为前缀的匹配
* 已经是否相等

> 由于for循环遍历匹配，所以当URLPattern变多时，匹配效率较低

2.3.3 response的参数处理和globalRoutes匹配

将response的参数存入finalParameters字典中，然后执行route的handlerBlock。

```objc
// configure the final parameters
        NSMutableDictionary *finalParameters = [NSMutableDictionary dictionary];
        [finalParameters addEntriesFromDictionary:response.parameters];
        [finalParameters addEntriesFromDictionary:parameters];
        [self _verboseLog:@"Final parameters are %@", finalParameters];
        
        didRoute = [route callHandlerBlockWithParameters:finalParameters];
```

使用 globalRoutes 进行匹配

```objc
// if we couldn't find a match and this routes controller specifies to fallback and its also not the global routes controller, then...
    if (!didRoute && self.shouldFallbackToGlobalRoutes && ![self _isGlobalRoutesController]) {
        [self _verboseLog:@"Falling back to global routes..."];
        didRoute = [[JLRoutes globalRoutes] _routeURL:URL withParameters:parameters executeRouteBlock:executeRouteBlock];
    }
```

如果匹配失败调用unmatchedURLHandler方法

```objc
// if, after everything, we did not route anything and we have an unmatched URL handler, then call it
    if (!didRoute && executeRouteBlock && self.unmatchedURLHandler) {
        [self _verboseLog:@"Falling back to the unmatched URL handler"];
        self.unmatchedURLHandler(self, URL, parameters);
    }
```


> 以上原理解析文章来源：https://www.jianshu.com/p/d24e9a7c8d4e、https://www.cnblogs.com/fakeCoder/p/7026211.html、https://www.jianshu.com/p/76aa39637ea6
