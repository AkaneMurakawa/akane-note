# Filter

## Filter

过滤器

用途：例如登录拦截等，权限拦截等

```java
@Slf4j
public class ReceiverFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        ServletRequest requestWrapper = null;
        if(request instanceof HttpServletRequest) {
            requestWrapper = new BodyReaderHttpServletRequestWrapper((HttpServletRequest) request);
        }
        if(null == requestWrapper) {
            chain.doFilter(request, response);
        } else {
            if (requestWrapper.getContentType().equalsIgnoreCase(MediaType.APPLICATION_FORM_URLENCODED_VALUE)) {
                Map params = requestWrapper.getParameterMap();
                if (!CollectionUtils.isEmpty(params)) {
                    Map<String, Object> paramsMap = new HashMap<>(params.size());
                    for (Map.Entry<String, String[]> entry : requestWrapper.getParameterMap().entrySet()) {
                        Object valueStr = (null != entry.getValue() && entry.getValue().length > 0) ? entry.getValue()[0] : null;
                        paramsMap.put(entry.getKey(), valueStr);
                    }
                    requestWrapper.setAttribute(ReceiverConst.REQUEST_API_BODY, JsonUtils.beanToJson(paramsMap));
                }
            } else {
                // 请求参数（过滤器将请求参数封装到request attribute）
                requestWrapper.setAttribute(ReceiverConst.REQUEST_API_BODY, StreamUtils.copyToString(requestWrapper.getInputStream(), Charset.defaultCharset()));
            }
            chain.doFilter(requestWrapper, response);
        }
    }

    @Override
    public void destroy() {

    }

}
@Configuration
public class ReceiverConfig implements WebMvcConfigurer {
    //还需要注入
    @Bean
    public FilterRegistrationBean filterRegist() {
        FilterRegistrationBean frBean = new FilterRegistrationBean();
        frBean.setFilter(new ReceiverFilter());
        frBean.addUrlPatterns("/*");
        frBean.setOrder(1);
        return frBean;
    }
}

// ------------------------------------------------------------------------------------------

// 另一种写法
@Component
@WebFilter(filterName = "MDCFilter", urlPatterns = "/*")
public class MDCFilter implements Filter {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
		// 通过，执行
        filterChain.doFilter(servletRequest, servletResponse);
    }
}
```



## HttpServletRequestWrapper

```java
public class BodyReaderHttpServletRequestWrapper extends HttpServletRequestWrapper {
    private final byte[] body;

    public BodyReaderHttpServletRequestWrapper(HttpServletRequest request)
            throws IOException {
        super(request);
        body = readBytes(request.getReader(), "utf-8");
    }

    @Override
    public BufferedReader getReader() {
        return new BufferedReader(new InputStreamReader(getInputStream()), 102400);
    }

    @Override
    public ServletInputStream getInputStream() {
        final ByteArrayInputStream bais = new ByteArrayInputStream(body);
        return new ServletInputStream() {

            @Override
            public boolean isFinished() {
                return false;
            }

            @Override
            public boolean isReady() {
                return false;
            }

            @Override
            public void setReadListener(ReadListener listener) {

            }

            @Override
            public int read() {
                return bais.read();
            }
        };
    }

    /**
     * 通过BufferedReader和字符编码集转换成byte数组
     * @param br
     * @param encoding
     * @return
     * @throws IOException
     */
    private byte[] readBytes(BufferedReader br,String encoding) throws IOException{
        String str;StringBuilder retStr= new StringBuilder();
        while ((str = br.readLine()) != null) {
            retStr.append(str);
        }
        if (StringUtils.isNotBlank(retStr.toString())) {
            return retStr.toString().getBytes(Charset.forName(encoding));
        }
        return new byte[0];
    }

}
```