#### Endpoint和Operation 使用

##### Endpoint注解使用

id为路径 /actuator/{id}

##### Operation 对应的HTTP请求方法

WriteOperation - Post

DeleteOperation - Delelte

ReadOperation - Get

##### 例子

动态配置更新-  org.springframework.cloud.endpoint.RefreshEndpoint



#### 动态配置更新

HTTP修改Env环境变量的URL：/actuator/env/{param}

具体的Operation操作

org.springframework.cloud.context.environment.WritableEnvironmentEndpointWebExtension

再改变属性后回发送EnvironmentChangeEvent事件进行通知



触发 org.springframework.cloud.context.properties.ConfigurationPropertiesRebinder 重新绑定Properties文件

org.springframework.cloud.context.properties.ConfigurationPropertiesRebinder#rebind(java.lang.String)