# spring 

## SpringMVC 如何获取 “请求的url”.

rulesetid=23

q/v1/ruleset/{rulesetId}

1. HttpServletRequest#getRequestURI: q/v1/ruleset/23
2. org.springframework.web.servlet.HandlerMapping#BEST_MATCHING_PATTERN_ATTRIBUTE: q/v1/ruleset/{rulesetId}

https://stackoverflow.com/questions/57241214/getting-the-template-of-url-that-has-been-hit


