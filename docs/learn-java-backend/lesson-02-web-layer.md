# 第 2 课　Web 层：一次请求的完整链路

## 本课目标

1. 浏览器地址栏敲下 `localhost:8080/owners/1` 之后，服务器内部依次发生了什么？
2. Spring MVC 的一堆注解分别在什么位置、解决什么问题？
3. 表单提交失败时，用户填过的内容为什么还在？
4. 为什么保存成功后要 `redirect:`，而不是直接返回页面？
5. 同一个 `VetController` 为什么既能返回网页、又能返回 JSON？

预计用时 4-6 小时（含 5 个实验）。本课是零基础最容易卡住的地方，请务必把实验做完。

## 1. 先想清楚：没有框架的时候要做什么

裸写 Servlet 处理一个请求，你要自己完成：

1. 解析 URL，判断该由哪段代码处理（`if (path.equals("/owners/1"))` 这种）
2. 把 URL 里的 `1` 抠出来，转成 `int`
3. 把请求参数绑定成 Java 对象
4. 校验参数，收集错误信息
5. 调用业务代码，拿到数据
6. 把数据塞进 HTML 模板渲染
7. 写响应

Spring MVC 把这七件事全部抽象成了固定的"零件"，你只需要写业务那一步。**理解 Web 层的关键，就是搞清楚这些零件叫什么、按什么顺序工作。**

## 2. DispatcherServlet：前端控制器

整个应用对外只有**一个 Servlet**，叫 `DispatcherServlet`（由自动配置注册，你从没写过它）。所有请求都先到它手上，再由它分派。

```
浏览器
  |  GET /owners/1
  v
Tomcat（内嵌）
  |
  v
DispatcherServlet            <- Spring MVC 的唯一入口
  |
  |-- 1. HandlerMapping：这个 URL 归谁处理？-> OwnerController.showOwner
  |-- 2. HandlerAdapter：准备参数（解析 @PathVariable 等）并调用方法
  |-- 3. 收到返回值 ModelAndView（视图名 + 数据）
  |-- 4. ViewResolver：把逻辑视图名 "owners/ownerDetails"
  |                     解析成 templates/owners/ownerDetails.html
  v
Thymeleaf 渲染 HTML -> 写回响应
```

**逻辑视图名**是一个很重要的概念：Controller 返回的 `"owners/ownerDetails"` 不是文件路径，只是一个名字，真正的路径由模板位置和前后缀配置推导。所以改模板目录不需要改 Java 代码。

## 3. 注解地图

先建立一张全局地图，再去读代码就不会迷路。

| 注解 | 写在哪里 | 作用 | 本项目例子 |
| --- | --- | --- | --- |
| `@Controller` | 类 | 声明这是处理 Web 请求的组件 | 所有控制器 |
| `@RequestMapping` | 类或方法 | 路径前缀 / 通用映射 | `PetController` 类上的 `/owners/{ownerId}` |
| `@GetMapping` / `@PostMapping` | 方法 | 限定 HTTP 方法 | 几乎所有方法 |
| `@PathVariable` | 参数 | 取 URL 路径里的值 | `@PathVariable("ownerId") int ownerId` |
| `@RequestParam` | 参数 | 取查询串或表单里的值 | `@RequestParam(defaultValue = "1") int page` |
| `@ModelAttribute` | 方法或参数 | 准备表单对象 / 绑定表单字段 | `findOwner`、`findPet` |
| `@InitBinder` | 方法 | 配置参数绑定规则 | 禁止绑定 `id` 字段 |
| `@Valid` | 参数 | 触发校验 | `@Valid Owner owner` |
| `BindingResult` | 参数 | 接收校验结果（必须紧跟被校验对象） | `@Valid Owner owner, BindingResult result` |
| `RedirectAttributes` | 参数 | 跨重定向传递一次性提示信息 | `addFlashAttribute("message", ...)` |
| `@ResponseBody` | 方法 | 返回值直接当响应体（数据），不走模板 | `VetController.showResourcesVetList` |

## 4. 链路 A：GET /owners/1（查看详情）

教材文件：`src/main/java/org/springframework/samples/petclinic/owner/OwnerController.java`

```java
@ModelAttribute("owner")
public Owner findOwner(@PathVariable(name = "ownerId", required = false) Integer ownerId) {
    return ownerId == null ? new Owner()
            : this.owners.findById(ownerId)
                .orElseThrow(() -> new IllegalArgumentException("Owner not found with id: " + ownerId + ...));
}

@GetMapping("/owners/{ownerId}")
public ModelAndView showOwner(@PathVariable("ownerId") int ownerId) {
    ModelAndView mav = new ModelAndView("owners/ownerDetails");
    Optional<Owner> optionalOwner = this.owners.findById(ownerId);
    Owner owner = optionalOwner.orElseThrow(() -> new IllegalArgumentException(...));
    mav.addObject(owner);
    return mav;
}
```

执行顺序：

1. Tomcat 收到请求，交给 `DispatcherServlet`。
2. 请求先经过拦截器链，其中 `LocaleChangeInterceptor`（第 1 课讲的 `WebConfiguration` 注册的）会检查有没有 `?lang=xx`。
3. `HandlerMapping` 匹配到 `@GetMapping("/owners/{ownerId}")`。
4. **在调用 `showOwner` 之前**，Spring 先执行本控制器里所有 `@ModelAttribute` 方法，也就是 `findOwner`——所以数据库会被查**两次**（一次在 `findOwner`，一次在 `showOwner`）。这是本项目一个可以优化的点，你读代码时要能发现它。
5. `@PathVariable` 把 URL 中的 `1` 转成 `int` 传给方法。类型不对会直接 400。
6. 调用 `owners.findById(1)`，Spring Data 代理生成 SQL 去查库。
7. 查不到就抛 `IllegalArgumentException`，被容器捕获后渲染 `templates/error.html`。这就是 `/owners/99999` 报错页的来历。
8. 查到则返回 `ModelAndView`：视图名 `owners/ownerDetails` + 模型里的 `owner` 对象（注意 `addObject(owner)` 不带 key 时，key 由类型推断为 `owner`）。
9. `ViewResolver` 解析模板，Thymeleaf 把 `${owner.firstName}` 之类的占位符替换成真实值。
10. HTML 写回浏览器。全程 Controller 只有一个方法体是你写的。

## 5. 链路 B：GET /owners?lastName=F（搜索）

```java
@GetMapping("/owners")
public String processFindForm(@RequestParam(defaultValue = "1") int page, Owner owner, BindingResult result,
        Model model) {
    String lastName = owner.getLastName();
    ...
    Page<Owner> ownersResults = findPaginatedForOwnersLastName(page, lastName);
    if (ownersResults.isEmpty()) {
        result.rejectValue("lastName", "notFound", "not found");
        return "owners/findOwners";
    }
    if (ownersResults.getTotalElements() == 1) {
        owner = ownersResults.iterator().next();
        return "redirect:/owners/" + owner.getId();
    }
    return addPaginationModel(page, model, ownersResults);
}

private Page<Owner> findPaginatedForOwnersLastName(int page, String lastname) {
    int pageSize = 5;
    Pageable pageable = PageRequest.of(page - 1, pageSize);
    return owners.findByLastNameStartingWith(lastname, pageable);
}
```

四个新知识点：

- 参数 `Owner owner` 前面**没有注解**，但依然会被当作 `@ModelAttribute` 处理：Spring 会根据 URL 上的查询参数 `?lastName=F` 自动 new 一个 `Owner` 并调用 `setLastName("F")`。这叫**表单对象绑定**。
- `@RequestParam(defaultValue = "1") int page`：URL 没带 `page` 参数时默认第 1 页。
- `PageRequest.of(page - 1, pageSize)`：URL 里页码从 1 开始（给人看），Spring Data 从 0 开始（给机器用），所以要减 1。这是新手最常踩的坑之一。
- 三条分支对应三种返回：查不到 → 回到查找页并带错误；只查到 1 条 → `redirect:` 跳到详情页；多条 → 渲染列表页。

**为什么"只查到 1 条"要重定向？** 因为重定向会让浏览器地址栏变成干净、可收藏、可刷新的 `/owners/5`。直接返回页面的话，刷新就会重放查询请求。

## 6. 链路 C：POST /owners/new（提交表单）

```java
@PostMapping("/owners/new")
public String processCreationForm(@Valid Owner owner, BindingResult result, RedirectAttributes redirectAttributes) {
    if (result.hasErrors()) {
        redirectAttributes.addFlashAttribute("error", "There was an error in creating the owner.");
        return VIEWS_OWNER_CREATE_OR_UPDATE_FORM;
    }
    this.owners.save(owner);
    redirectAttributes.addFlashAttribute("message", "New Owner Created");
    return "redirect:/owners/" + owner.getId();
}
```

三条必须记住的规则：

1. **`BindingResult` 必须紧跟在被校验对象后面。** 写成 `(BindingResult result, @Valid Owner owner)` 会直接抛异常。这是硬性规则，不是风格问题。
2. **校验失败时返回的是视图名（转发），不是重定向。** 因为只有转发才能保留 `owner` 对象里的用户输入，页面上的表单才能回显。重定向会丢掉这些数据。
3. **校验成功时用 `redirect:`。** 这叫 PRG（Post/Redirect/Get）模式：提交用 POST，成功后用 GET 打开结果页。这样用户按 F5 刷新时不会重复提交一次创建请求。

`RedirectAttributes.addFlashAttribute(...)` 是 PRG 的配套工具：它把数据放进"闪存"（本质是 Session），只在**下一次请求**可见，读完即删。所以重定向后的页面能显示"New Owner Created"，但刷新一次就消失了。

另外注意类上的 `@InitBinder`：

```java
@InitBinder
public void setAllowedFields(WebDataBinder dataBinder) {
    dataBinder.setDisallowedFields("id", "*.id");
}
```

这是在参数绑定的入口处做安全防护：即使前端伪造了 `id=123` 字段，也不会被绑定到对象上。**凡是涉及主键、权限标记这类字段，都应该在这里挡掉。**

## 7. 同一份数据，页面还是 JSON？

`src/main/java/org/springframework/samples/petclinic/vet/VetController.java` 里有鲜明对比：

```java
@GetMapping("/vets.html")
public String showVetList(@RequestParam(defaultValue = "1") int page, Model model) {
    Page<Vet> paginated = findPaginated(page);
    return addPaginationModel(page, paginated, model);   // 返回视图名 -> 渲染页面
}

@GetMapping({ "/vets" })
public @ResponseBody Vets showResourcesVetList() {
    Vets vets = new Vets();
    vets.getVetList().addAll(this.vetRepository.findAll());
    return vets;                                          // 返回对象 -> 序列化成 JSON
}
```

Handler 方法的返回值一共就这三种形态，务必分清：

| 返回值 | 含义 | 例子 |
| --- | --- | --- |
| `String` | 逻辑视图名；带 `redirect:` 前缀则是重定向 | `return "owners/ownersList"` |
| `ModelAndView` | 视图名 + 数据一起返回 | `OwnerController.showOwner` |
| 对象 + `@ResponseBody` | 不走模板，直接把对象写成响应体（JSON） | `VetController.showResourcesVetList` |

这也解释了为什么 `Vets` 这个"包装类"会存在——注释里写得很直白：返回一个对象比返回集合更好做 JSON 映射。

## 8. 动手实验

### 实验 1（20 分钟）：写你的第一个路由

新建 `src/main/java/org/springframework/samples/petclinic/system/HelloController.java`：

```java
package org.springframework.samples.petclinic.system;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
class HelloController {

    @GetMapping("/hello")
    public @ResponseBody String hello() {
        return "hello";
    }

}
```

重启访问 `localhost:8080/hello`。然后把 `@ResponseBody` 去掉，再访问一次，观察报错（Spring 会去 `templates/hello.html` 找模板，找不到就 500）。

观察点：**同一个 `String` 返回值，有没有 `@ResponseBody`，含义完全不同。**

### 实验 2（15 分钟）：看 @ModelAttribute 的执行时机

在 `OwnerController.findOwner` 里加一行 `System.out.println("findOwner: " + ownerId);`。

重启后分别访问 `/owners/new` 和 `/owners/1`，看控制台。

观察点：访问 `/owners/new` 时 `ownerId` 是 `null`（路径里没有这个变量，所以 `required = false`）；访问 `/owners/1` 时它被调用了。同时你会发现这个方法是"每个请求都跑一次"，这正是下面实验 3 要利用的特性。

### 实验 3（15 分钟）：破坏 BindingResult 的位置

把 `processCreationForm` 的参数顺序改成：

```java
public String processCreationForm(BindingResult result, @Valid Owner owner, RedirectAttributes redirectAttributes)
```

重启，提交一次新建表单，观察异常信息（大意是 `BindingResult` 与被校验对象之间不能有其他参数）。

改回来。

观察点：框架的"约定"是靠抛异常来强制执行的，**读懂异常信息本身就是核心能力**。

### 实验 4（20 分钟）：亲手验证 PRG 的价值

把 `processCreationForm` 成功分支的 `return "redirect:/owners/" + owner.getId();` 临时改成直接返回视图：

```java
return VIEWS_OWNER_CREATE_OR_UPDATE_FORM;
```

重启，新建一个主人。提交成功后按 F5 刷新——浏览器会弹"是否重新提交表单"的提示。

改回 `redirect:`，再试一次，刷新就不会有提示了。

观察点：**这就是 PRG 模式存在的全部理由。** 生产环境的重复下单、重复扣款，很多都是漏了这一步。

### 实验 5（10 分钟）：页面与接口的对照

浏览器访问 `localhost:8080/vets.html`（页面），再用 `curl localhost:8080/vets` 或浏览器开发者工具的 Network 面板看 `/vets` 的响应（JSON）。

观察点：同一份数据、同一个 Controller，只因为返回值写法不同，就分别成了"给人看的页面"和"给程序调的接口"。这是前后端分离的基础。

## 9. 自测题

1. `DispatcherServlet` 在请求处理中扮演什么角色？它下面有哪几个关键零件？
2. 逻辑视图名 `"owners/ownerDetails"` 最终是怎么变成 HTML 文件的？
3. `@PathVariable` 和 `@RequestParam` 分别从哪里取值？
4. 为什么 `BindingResult` 必须紧跟被校验对象？
5. 校验失败时返回视图名、成功时返回 `redirect:`，这个不对称设计的理由是什么？
6. `addFlashAttribute` 的数据为什么刷新页面后就没了？
7. `@InitBinder` 里禁止绑定 `id` 字段，防的是什么问题？
8. `PageRequest.of(page - 1, pageSize)` 里的减 1 是为了对齐什么差异？

## 10. 本课要记住的五句话

1. 所有请求都先到 `DispatcherServlet`，由它分派到具体方法；你写的只是链路中最中间那一段。
2. Controller 的返回值只有三种形态：视图名、`ModelAndView`、`@ResponseBody` 对象。
3. 表单提交遵循 PRG：失败转发回显，成功重定向，用 `RedirectAttributes` 传递一次性提示。
4. `BindingResult` 的位置是硬性规则，`@InitBinder` 是安全防线。
5. URL 里的页码从 1 开始，Spring Data 从 0 开始，这个减 1 不能忘。

## 附：自测题答案

1. 它是 Spring MVC 唯一的前端控制器，所有请求的统一入口；关键零件是 `HandlerMapping`（找处理器）、`HandlerAdapter`（准备参数并调用）、`ViewResolver`（把视图名解析成模板）。
2. `ViewResolver` 用配置的前后缀拼出真实路径 `classpath:/templates/owners/ownerDetails.html`，再交给 Thymeleaf 渲染。
3. `@PathVariable` 取 URL 路径中由 `{}` 占位的那一段；`@RequestParam` 取查询串（`?a=b`）或表单提交的字段。
4. 因为框架是按"参数顺序"把校验结果与被校验对象配对的，中间插入其他参数会让它无法确定该把结果交给谁，于是直接抛异常。
5. 失败时要用用户刚填的数据回显，必须转发；成功时要用干净的 GET 地址展示结果并防止重复提交，必须重定向。
6. 它的底层是 Session 中的一次性属性，被读取一次后立即移除，所以只在下一次请求可见。
7. 防止前端伪造 `id` 等字段，造成越权修改他人数据或篡改主键。
8. 对齐"URL 中页码从 1 开始（便于人阅读）"和"Spring Data 页码从 0 开始"这两套约定。
