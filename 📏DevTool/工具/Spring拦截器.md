# Spring

## 拦截器HandlerInterceptor

```
public class IdempotentInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {return false;}
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {}
}    
 
@Configuration
public class ReceiverConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 拦截的请求
        registry.addInterceptor(new IdempotentInterceptor()).addPathPatterns("/xx/xxx");
    }
}    
     
```