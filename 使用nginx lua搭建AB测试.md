首先推荐一本Nginx+Lua开发的[电子书](http://wiki.jikexueyuan.com/project/nginx-lua/development-environment.html)

Nginx Lua是由国人开发的一个Nginx模块，核心开发者[微博agentzh](http://weibo.com/u/1834459124),支持Lua 5.1或者LuaJIT（2.0/2.1）。需要注意的是这个模块中的Lua代码都是已非阻塞模式运行的，这样就可以保证在Lua里使用Redis，Mysql，Memcached是没有问题的。

为了方便开发者开发web，[微博agentzh](http://weibo.com/u/1834459124)开发了[openresty](https://openresty.org/)这个nginx lua web框架，我们就要使用这个框架来完成我们的工作，因为它对nginx的很多东西提供了更高级的封装，也提供了更多高级的web开发api，避免了我们直接在lua拼接http cookie，header等原始的写法。具体的环境配置我就不啰嗦了，agentzh提供了一个bundle，建议直接从GitHub上clone下来编译，这个bundle包含了nginx，lua以及很多常用的nginx module。

AB测试这里我使用的方法是根据cookie中的值来决定分配到哪个服务器，如果第一次访问，会随机生成一个值，并将这个值写入cookie。代码其实很简单

```lua
local ck = require("resty.cookie")
local cookie = ck:new()
local abtest = cookie:get("abtest")
if abtest == nil then
  math.randomseed(tostring(os.time()):reverse():sub(1, 6))
  r = math.floor(math.random()*10)
  if r % 2 == 0 then
    if ngx.req.get_headers()["Cookie"] then
      ngx.header["Set-Cookie"]="abtest=1; path=/;"..ngx.req.get_headers()["Cookie"]
    else
      ngx.header["Set-Cookie"]="abtest=1; path=/;"
    end
    ngx.log(ngx.ERR, "set abtest=1 go to webv2")
    ngx.redirect("http://www.xxx.com")
  else
    if ngx.req.get_headers()["Cookie"] then
      ngx.header["Set-Cookie"]="abtest=0; path=/;"..ngx.req.get_headers()["Cookie"]
    else
      ngx.header["Set-Cookie"]="abtest=0; path=/;"
    end
    ngx.log(ngx.ERR, "set abtest=0 go to webv1")
    ngx.redirect("http://www.xxx.com")
  end
else
  if abtest == "1" then
    ngx.log(ngx.ERR, "abtest=1 go to webv2")
    ngx.exec("@sitev2")
  else
    ngx.log(ngx.ERR, "abtest=0 go to webv1")
    ngx.exec("@sitev1")
  end
end
```

另外新浪技术保障部开源了一套自己的Nginx Lua AB测试框架，包含了基于ip，user id等分流策略，还有一套web管理界面，方便随时修改策略。[GitHub地址](https://github.com/SinaMSRE/ABTestingGateway)。我后面会写几篇博客来介绍新浪的代码，也推荐大家尽量使用新浪的这套解决方案，非常的赞。