DispatcherServlet.doDispatch(HttpServletRequest, HttpServletResponse)中的代码`processedRequest = checkMultipart(request);`对request检查了其是否包含文件内容，有的话，就得做进一步的封装了。不过首先，我们先来了解一下RequestFacade的组成。

## Request组成

不包含文件情况下：

- RequestFacade

  - request：connector.

    - coyoteRequest：coyote.

      - parameters：请求中不具有特殊含义的文本数据k-v键值对。

    - inputBuffer：InputBuffer

      inputStream：CoyoteInputStream

      reader：CoyoteReader

      上述三个字段是用来读取JSON数据的，只不过，request有选择性地选择字节流或字符流读取，至于多出来的一个，我也不知道是干什么的，只不过是读取流，就加入进来了。

    - others

包含文件情况下：

- StandardMultipartHttpServletRequest
  - multipartParameterNames：Set\<String> 。form-data中文本数据记录的key集合。
  - multipartFiles：MultiValueMap<String, MultipartFile> 。可能存在同key文件数据。MultipartFile：
    - String：param-name
    - MultipartFile：StandardMultipartHttpServletRequest.StandardMultipartFile。
      - part：ApplicationPart。应该是提供数据流的
        - fileItem：DiskFileItem
        - File
      - filename：提供的文件名
  - RequestFacade



## Multipart/form-data Request封装处理

multipartResolver：StandardServletMultipartResolver

```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
   // 匹配request的content-type是不是以“multipart/”前缀（严格情况下以multipart/form-data开头）。
   if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
   		...........................
         // 提供多媒体类型的Request，封装原本request。
   		return this.multipartResolver.resolveMultipart(request);
 		...........................
   }
   // 不是多媒体类型request，返回原本request。
   return request;
}
```

StandardServletMultipartResolver.resolveMultipart(HttpServletRequest)

```java
// 提供StandardMultipartHttpServletRequest。
public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
   return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
}
```

StandardMultipartHttpServletRequest.parseRequest(HttpServletRequest)

以图片为描述依据：

<img src="picture/2024-02-29 15_37_36-localhost_9090_info_ggl_hello=20210101&hello=sdfdsf - My Workspace.png" style="zoom:50%;" />

> 不管是不是只有文本数据，都会被封装。

```java
private void parseRequest(HttpServletRequest request) {
   try {
      // 元素实际类是ApplicationPart，每个都记录了form-data数据行。
      // multipart/form-data中文本数据同时存入了coyote.request.parameters。
      Collection<Part> parts = request.getParts();
      this.multipartParameterNames = new LinkedHashSet<>(parts.size());
      MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<>(parts.size());
      for (Part part : parts) {
         String headerValue = part.getHeader(HttpHeaders.CONTENT_DISPOSITION);
         ContentDisposition disposition = ContentDisposition.parse(headerValue);
         String filename = disposition.getFilename();
         if (filename != null) {
            // 文件数据收集
            if (filename.startsWith("=?") && filename.endsWith("?=")) {
               filename = MimeDelegate.decode(filename);
            }
            // 每个Part包装 - StandardMultipartFile(part, filename)
            files.add(part.getName(), new StandardMultipartFile(part, filename));
         }
         else {
            // multipart/form-data中的文本数据的key收集
            this.multipartParameterNames.add(part.getName());
         }
      }
      // 为this.multipartFiles提供LinkedMultiValueMap(files)。
      // this.multipartFiles - LinkedMultiValueMap(MultiValueMap<param-name, StandardMultipartFile(fileName, ApplicationPart(InputStream))>)
      setMultipartFiles(files);
   }
   catch (Throwable ex) {
      handleParseFailure(ex);
   }
}
```

