# 自定义URL的限流响应内容

```
import com.alibaba.csp.sentinel.adapter.servlet.callback.UrlBlockHandler;
import com.alibaba.csp.sentinel.adapter.servlet.callback.WebCallbackManager;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityException;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeException;
import com.alibaba.csp.sentinel.slots.block.flow.FlowException;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowException;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;
@Configuration
public class CustomUrlBlockHandlerConfigurer {
    @Bean
    CustomUrlBlockHandler customUrlBlockHandler() {
        CustomUrlBlockHandler customUrlBlockHandler = new CustomUrlBlockHandler();
        WebCallbackManager.setUrlBlockHandler(new CustomUrlBlockHandler());
        return customUrlBlockHandler;
    }

    private static class CustomUrlBlockHandler implements UrlBlockHandler {
        @Override
        public void blocked(HttpServletRequest request, HttpServletResponse response, BlockException ex) throws IOException {
            response.setCharacterEncoding("utf-8");
            response.setContentType("application/json; charset=utf-8");
            String message = "";
            if (ex instanceof FlowException) {
                message = "触发流量控制";
            } else if (ex instanceof DegradeException) {
                message = "触发熔断降级";
            } else if (ex instanceof ParamFlowException) {
                message = "触发热点参数限流";
            } else if (ex instanceof AuthorityException){
                message = "触发权限规则";
            }
            // 自定义返回客户端内容
            APIResponse apiResponse = APIResponse.buildFailResult(message);
            PrintWriter out = response.getWriter();
            out.write(JsonUtils.beanToJson(apiResponse));
            out.close();
        }
    }
}


```

