- GET在浏览器回退是无害的，而POST会再次提交请求
- GET请求会被浏览器主动cache,而POST不会，除非手动设置
- GET请求只能进行URL编码，而POST支持多种编码
- GET请求参数会被完整保留在浏览器历史记录中，而POST中的参数不会被保留
- GET请求在URL中传送参数是有大小限制的，不能大于2KB,而POST可以说没有
- GET只接受ASCII字符，而POST没有限制
- GET参数直接暴露在URL上，而POST将数据放在request body中

