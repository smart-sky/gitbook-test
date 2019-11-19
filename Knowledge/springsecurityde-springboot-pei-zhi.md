


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
                //异常处理配置
                .exceptionHandling()
                    //权限不足时处理
                    .accessDeniedHandler(deniedHandler);
    }
}
```

###3.获取用户详情，验证用户

继承UserDetailsService

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


其中的Hr类是需要实现UserDetails

```java
public class Hr implements UserDetails {
    private Long id;
    private String name;
    private String phone;
    private String telephone;
    private String address;
    private boolean enabled;
    private String username;
    private String password;
    private String remark;
    private List<Role> roles;
    private String userface;




    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
       List<GrantedAuthority> authorities = new ArrayList<>();
       for(Role role :roles){
           authorities.add(new SimpleGrantedAuthority(role.getName()));
        }
        return authorities;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return enabled;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPhone() {
        return phone;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }

    public String getTelephone() {
        return telephone;
    }

    public void setTelephone(String telephone) {
        this.telephone = telephone;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getRemark() {
        return remark;
    }

    public void setRemark(String remark) {
        this.remark = remark;
    }

    public List<Role> getRoles() {
        return roles;
    }

    public void setRoles(List<Role> roles) {
        this.roles = roles;
    }

    public String getUserface() {
        return userface;
    }

    public void setUserface(String userface) {
        this.userface = userface;
    }
}

```
查询库的语句是
```xml
<select id="loadUserByUsername" resultMap="lazyLoadRoles">
	select * from hr where username = #{username}
</select>
<select id="getRolesByHrId" resultType="org.sang.bean.Role">
	select r.* from hr_role h,role r where h.rid=r.id and h.hrid =#{id}
</select>
<resultMap id="lazyLoadRoles" type="org.sang.bean.Hr" extends="BaseResultMap">
	<collection property="roles" ofType="org.sang.bean.Hr" select="org.sang.mapper.HrMapper.getRolesByHrId"
column="id">
  	</collection>
</resultMap>
```

###4.配置查询页面所需要的角色
```java

/**
 * 该类的主要功能就是通过当前的请求地址，获取该地址需要的用户角色
 */

@Component
public class CustomMetadataSource implements FilterInvocationSecurityMetadataSource {

    @Autowired
    MenuService menuService;
    AntPathMatcher antPathMatcher = new AntPathMatcher();
    @Override
    public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
        String requestUrl = ((FilterInvocation) o).getRequestUrl();
        List<Menu> allMenu = menuService.getAllMenu();
        for (Menu menu:allMenu){
            if (antPathMatcher.match(menu.getUrl(),requestUrl) && menu.getRoles().size()>0){
                List<Role> roles = menu.getRoles();
                int size = roles.size();
                String[] values = new String[size];
                for (int i = 0;i<size;i++){
                    values[i] = roles.get(i).getName();
                }
                return SecurityConfig.createList(values);
            }
        }
        return SecurityConfig.createList("ROLE_LOGIN");

    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return false;
    }
}
```





###5.查询帐号所拥有的角色
```java
@Component
public class UrlAccessDecisionManager implements AccessDecisionManager {

//    其中第一个参数中保存了当前登录用户的角色信息
//    ，第三个参数则是UrlFilterInvocationSecurityMetadataSource中的getAttributes方法传来的，表示当前请求需要的角色（可能有多个）。
    @Override
    public void decide(Authentication auth, Object o, Collection<ConfigAttribute> cas) {
        Iterator<ConfigAttribute> iterator = cas.iterator();
        while (iterator.hasNext()){

            ConfigAttribute ca = iterator.next();

            //当前请求需要得权限
            String needRole = ca.getAttribute();
            if ("ROLE_LOGIN".equals(needRole)){
                if (auth instanceof AnonymousAuthenticationToken){
                    throw new BadCredentialsException("未登陆");
                }else{
                    return;
                }
            }
            //当前用户所具有得权限
            Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
            for (GrantedAuthority authority : authorities){
                if (authority.getAuthority().equals(needRole)){
                    return;
                }
            }
        }
        throw new AccessDeniedException("权限不足");
    }

    @Override
    public boolean supports(ConfigAttribute configAttribute) {
        return false;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return false;
    }
}

```



### 6.无权限异常处理

```java
@Component
public class AuthenticationAccessDeniedHandler implements AccessDeniedHandler {


    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse resp, AccessDeniedException e) throws IOException{
        resp.setStatus(HttpServletResponse.SC_FORBIDDEN);
        resp.setContentType("application/json;charset=UTF-8");
        PrintWriter out = resp.getWriter();
        RespBean error =  RespBean.error("权限不足，请联系管理员!");
        out.write(new ObjectMapper().writeValueAsString(error));
        out.flush();
        out.close();
    }
}
```








##内容参考来源：
1.https://www.jianshu.com/p/4cee4b19ec40
2.https://blog.csdn.net/weixin_42849689/article/details/89957823
3.代码例子取之git地址：https://github.com/lenve/vhr