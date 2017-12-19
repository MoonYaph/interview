# cookie

cookie 是服务器发送到用户浏览器并保存在浏览器上的数据，它会在浏览器下一次发起请求时被携带发送到服务器上。比较典型的，可以用来确定两次请求是否来自同一个浏览器，从而能够确认和保持用户的登录状态

cookie 主要用在三个方面：

* 会话状态管理(用户登录状态，购物车，游戏分数和其他需要记录的信息)
* 个性化设置(用户自定义设置、主题等)
* 浏览器行为跟踪(跟增分析用户行为)

第一次访问网站的时候，浏览器发出请求，服务器相应请求后，会将 cookie 放入到相应请求中，在浏览器第二次发请求的时候，会把 cookie 带过去，服务器会辨别用户身份，服务器也可以修改 cookie 的内容

### cookie 属性

* name cookie 名字
* value cookie 的值
* domain cookie 的域名，如果没有设置，会自动绑定到执行语句的当前域
* path 匹配的 web 路由 默认'/'
* cookie 的有效期:expires 或者 maxAge，maxAge 优先级高，maxAge 以秒为单位，为负数时，表示为临时存储，不会生成 cookie，存储在浏览器内存中，一旦关闭浏览器，cookie 就会消失，maxAge 为 0 时，删除 cookie，正数时，会在 n 秒后删除 cookie
* secure 安全，当为 true 时，cookie 只在 https 和 ssl 安全协议下传输
* httponly 设置为 true，前端无法获取 cookie，可以有效防止 xss 攻击
