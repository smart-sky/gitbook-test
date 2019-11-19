#springSecurity的springboot配置

##概述
  首先需要的基础springBoot环境。
  
  
##步骤
###1.包依赖

```xml
        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
```

###2.总配置代码

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig  extends WebSecurityConfigurerAdapter {

    @Autowired
    HrService hrService;

    @Autowired
    CustomMetadataSource metadataSource;

    @Autowired
    UrlAccessDecisionManager urlAccessDecisionManager;

    @Autowired
    AuthenticationAccessDeniedHandler deniedHandler;

    @Override//AuthenticationManagerBuilder用来配置全局的认证相关的信息
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        //用户明细查找和密码的加密方式
        //AuthenticationProvider和UserDetailsService，前者是认证服务提供者，后者是认证用户（及其权限）
        auth.userDetailsService(hrService).passwordEncoder(new BCryptPasswordEncoder());
    }

    /**
     * WebSecurity 全局请求忽略规则配置（比如说静态文件，比如说注册页面）、全局HttpFirewall配置、是否debug配置、全局SecurityFilterChain配置、privilegeEvaluator、expressionHandler、securityInterceptor。
     * @param web
     * @throws Exception
     */
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/index.html","/static/**","/login_p","/favicon.ico");
    }

    /**
     * 这段配置，我认为就是配置Security的认证策略, 每个模块配置使用and结尾。
     * @param http
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                //authorizeRequests()配置路径拦截，表明路径访问所对应的权限，角色，认证信息。
                .authorizeRequests()
                    .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                        @Override
                        public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                            //设置页面需要的角色集合配置
                            o.setSecurityMetadataSource(metadataSource);
                            //用户所拥有的角色
                            o.setAccessDecisionManager(urlAccessDecisionManager);
                            return o;
                        }
                    })
                    .and()
                //formLogin()对应表单认证相关的配置
                .formLogin()
                    //配置登陆页面，但是在本项目没用
                    .loginPage("/login_p")
                    //用于匹配登陆请求
                    .loginProcessingUrl("/login")
                    .usernameParameter("username")
                    .passwordParameter("password")
                    //失败处理
                    .failureHandler(new AuthenticationFailureHandler() {
                        @Override
                        public void onAuthenticationFailure(HttpServletRequest req,
                                                            HttpServletResponse resp,
                                                            AuthenticationException e) throws IOException {
                            resp.setContentType("application/json;charset=utf-8");
                            RespBean respBean = null;
                            if (e instanceof BadCredentialsException || e instanceof UsernameNotFoundException){
                                respBean = RespBean.error("账户名或者密码输入错误！");
                            }else if (e instanceof LockedException){
                                respBean = RespBean.error("帐号被锁定，请联系管理员！");
                            }else if (e instanceof CredentialsExpiredException){
                                respBean = RespBean.error("密码过期，请联系管理员！");
                            }else if (e instanceof AccountExpiredException){
                                respBean = RespBean.error("帐号过期，请联系管理员！");
                            }else if (e instanceof DisabledException){
                                respBean = RespBean.error("帐号被禁用，请联系管理员！");
                            }else {
                                respBean = RespBean.error("登陆失败！");
                            }
                            resp.setStatus(401);
                            ObjectMapper om = new ObjectMapper();
                            PrintWriter out = resp.getWriter();
                            //writeValueAsString方法可以把java对象转化成json字符串
                            out.write(om.writeValueAsString(respBean));
                            out.flush();
                            out.close();

                        }
                    })
                    .successHandler(new AuthenticationSuccessHandler() {
                        @Override
                        public void onAuthenticationSuccess(HttpServletRequest req,
                                                            HttpServletResponse resp,
                                                            Authentication auth) throws IOException {
                            resp.setContentType("application/json;charset=utf-8");
                            RespBean respBean = RespBean.ok("登录成功！", HrUtils.getCurrentHr());
                            ObjectMapper om = new ObjectMapper();
                            PrintWriter out = resp.getWriter();
                            out.write(om.writeValueAsString(respBean));
                            out.flush();
                            out.close();
                        }
                    })
                    .permitAll()
                    .and()
                //注销操作
                .logout()
                    .logoutUrl("/logout")
                    .logoutSuccessHandler(new LogoutSuccessHandler() {
                        @Override
                        public void onLogoutSuccess(HttpServletRequest req, HttpServletResponse resp, Authentication authentication) throws IOException, ServletException {
                            resp.setContentType("application/json;charset=utf-8");
                            RespBean respBean = RespBean.ok("注销成功！");
                            ObjectMapper om = new ObjectMapper();
                            PrintWriter out = resp.getWriter();
                            out.write(om.writeValueAsString(respBean));
                            out.flush();
                            out.close();
                        }
                    })
                    //允许任何人访问
                    .permitAll()
                    .and()
                //防护CSRF
                .csrf()
                    .disable()
                    .exceptionHandling()
                    .accessDeniedHandler(deniedHandler);
    }
}
```

###3.获取用户详情，验证用户
```java
@Service
@Transactional
public class HrService implements UserDetailsService {

    @Autowired
    HrMapper hrMapper;

    /**
     *  用于验证客户信息
     * @param s 帐号
     * @return
     * @throws UsernameNotFoundException
     */
    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        Hr hr = hrMapper.loadUserByUsername(s);
        if (hr == null){
            throw new UsernameNotFoundException("用户名不对");
        }
        return hr;
    }
    public List<Hr> getAllHr() {
        return hrMapper.getAllHr(null);
    }
}

```




##内容参考来源：
1.https://www.jianshu.com/p/4cee4b19ec40
2.https://blog.csdn.net/weixin_42849689/article/details/89957823
3.代码例子取之git地址：https://github.com/lenve/vhr