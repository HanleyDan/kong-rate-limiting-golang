# kong-rate-limiting-golang

### Characteristics
- A Kong rate limiting plug-in written in golang
- Rate limit supports concurrency
- Accurate rate limit
- Rate limiting configuration supports and and or matching rules for current limiting

### Environmental Requirements
- The kong version is 2.0 or higher to support the go plug-in (but the official website document says that version 2.0.5 fixes the problem of intermittently killing the go plug-in, so it is recommended to use version 2.0.5 and above. For update details, please refer to: https://github. com/Kong/kong/blob/master/CHANGELOG.md#205)

### Deployment Method 1：Use Docker(Clone the repo directly，Execute make related commands to build kong image）【If the code is cloned, you can use make run-kong-konga-pg to run the full set of kong, postgres and konga environments in the project root directory]】
- Pull Mirror
```
docker pull lampnick/kong-rate-limiting-plugin-golang:latest
```
- Run docker
```
docker run --rm --name kong-rate-limiting-plugin-golang \
    -e "KONG_LOG_LEVEL=info" \
    -e "KONG_NGINX_USER=root root" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    lampnick/kong-rate-limiting-plugin-golang:latest
```
- Test whether the plugin is loaded successfully
```
curl http://localhost:8001/ |grep --color custom-rate-limiting
```

### Deployment Method 2: server compilation and deployment (a small test, you can experience this plug-in in a few simple steps)
- clone this project to /etc/kong/
```
mkdir /etc/kong
cd /etc/kong
git clone https://github.com/lampnick/kong-rate-limiting-golang.git
```
- Modify the kong configuration file
```
plugins = bundled,custom-rate-limiting
go_plugins_dir = /etc/kong/plugins
go_pluginserver_exe = /usr/local/bin/go-pluginserver
```
- Build go-pluginserver
```
Execute go build github.com/Kong/go-pluginserver in go-pluginserver
The go-pluginserver file will be generated and copied to the /usr/local/bin directory
```
-  Compile the go plugin
```
go build -buildmode plugin custom-rate-limiting.go
```
- Put the generated .so file in the directory defined by go_plugins_dir (configured as /etc/kong/plugins above)
```.env
cp custom-rate-limiting.so /etc/kong/plugins/
```
- Restart kong
```
kong prepare && kong reload
```
- Configure the plugin in Konga
    - Configure route, service and other configurations by yourself
    - Konga json test configuration
        ```
        [{
            "type": "header,query,body",
            "key": "orderId",
            "value": "orderId1,orderId2,orderId3"
        }, {
            "type": "query",
            "key": "username",
            "value": "nick,jack,star"
        }]
        ```
    - konga placement:
    ![image](http://www.lampnick.com/wp-content/uploads/2020/09/kong-config.png)

- Test whether the request is normal and whether the rule takes effect (postman displays the header or the browser debug mode to view)
    ![image](http://www.lampnick.com/wp-content/uploads/2020/09/kong-post-header-show-2.png)

- siege pressure test to check whether the current limiting rule is in effect (returns the 429 status code, which is a current limited request. In the figure, there are 40 requests in total. The configured QPS is 20, but there are not 20 current limited because these requests are not. Restricted within 1S, spanning 1S time)
![image](http://www.lampnick.com/wp-content/uploads/2020/09/rate-limiting.png)

### Plug-in development process
1.Define a structure type to save the configuration file
```
The plug-in written in lua specifies how to read and verify the configuration data from the database and the Admin API through the schema. Since GO is a statically typed language, it needs to be defined with a configuration structure
type MyConfig struct {
    Path   string //The configuration here will be displayed when konga adds the plug-in
    Reopen bool
}
Public attributes will be filled with configuration data. If you want to use a different name in the database, you can use encoding/json plus tag
type MyConfig struct {
    Path   string `json:my_file_path`
    Reopen bool   `json:reopen`
}
```
2. Use New() to create an instance
```
Your go plugin must define a function named New to create an instance of this type and return an interface{} type
func New() interface{} {
    return &MyConfig{}
}
```
3. Add processing stage method
```
You can implement custom logic at various stages of the request's life cycle. For example, in the "access" phase, define a method named Access
func (conf *MyConfig) Access (kong *pdk.PDK) {
  ...
}
There are several phase methods you can implement custom logic as follows:
Certificate
Rewrite
Access
Preread
Log
```
4. Compile the go plugin
```
go build -buildmode plugin  custom-rate-limiting.go
```
5. Put the generated .so file in the directory defined by go_plugins_dir
```.env
cp custom-rate-limiting.so ../plugins/
```
6. Restart kong (smooth restart)
```
kong prepare && kong reload
```
