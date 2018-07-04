---
layout:     post
title:      "SpringBoot与JWT"
subtitle:   " \"Java Web Token的原理和SpringBoot的整合\""
date:       2018-06-22 12:00:00
author:     "echola"
header-img: "img/post-bg-2015.jpg"
tags:
    - SpringCloud
---


#### 原理
![image](http://note.youdao.com/yws/public/resource/9cb3656fb4795720f7a938c73b2543cd/xmlnote/FEB28D18DB624741946D80C6A9E13DA2/9418)  
##### 解决了哪些问题：
- 和session相比不用在server端保存一个连接客户端的会话
- 没有将生成的token保存到server端，所以即使每次请求是不同的服务端，也可以运行，实现了分布式。
#### JSON Web Token结构
结构： **xxxx.yyyy.zzzzz**
- Header  
Header包含两部分：1、token的类型，这里是JWT，2、所使用的hash算法（HMAC SHA256 或者 RSA）
```
{
  "alg": "HS256",
  "typ": "JWT"
}
```
然后Base64加密
- Payload  
主要包含三类声明：  
1. 预保留的声明(Reserved claims)，这类推荐但是不强制使用，包括： iss (issuer), exp (expiration time), sub (subject), aud (audience), and others 
2. 公共声明(Public claims),这类可以添加任何信息但不建议添加敏感信息。
3. 私有声明（Private claims），消费者和提供者之间共享的信息表明双方可以使用它们。
例如
```
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```
然后Base64加密
- Signature  
这一部分是将加密的header,加密的payload,秘钥（secret,存放到服务端）,用header中定义的加密算法进行加密。例如:
```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```
#### Java实现：
##### 使用组件
```
    <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt</artifactId>
			<version>0.7.0</version>
	</dependency>
```
1. 用户名密码登录到后台，用secret生成一个token  
**Controller登录相关代码**

```
  @RequestMapping(value = "${jwt.route.authentication.path}", method = RequestMethod.POST)
    public ResponseEntity<?> createAuthenticationToken(@RequestBody JwtAuthenticationRequest authenticationRequest, Device device) throws AuthenticationException {

        // Perform the security
        final Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        authenticationRequest.getUsername(),
                        authenticationRequest.getPassword()
                )
        );
        SecurityContextHolder.getContext().setAuthentication(authentication);

        // Reload password post-security so we can generate token
        final UserDetails userDetails = userDetailsService.loadUserByUsername(authenticationRequest.getUsername());
        final String token = jwtTokenUtil.generateToken(userDetails, device);

        // Return the token
        return ResponseEntity.ok(new JwtAuthenticationResponse(token));
    }
```

**生成token**:

```
  private String doGenerateToken(Map<String, Object> claims, String subject, String audience) {
        final Date createdDate = new Date();
        final Date expirationDate = calculateExpirationDate(createdDate);

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)//用户名
                .setAudience(audience)
                .setIssuedAt(createdDate)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }
```

2. 用户后续每次请求时将token放在header中传到后台
3. 后台filter解析token来获取用户名等需要的信息  

解密token:

```
    private Claims getClaimsFromToken(String token) {
        Claims claims;
        try {
            claims = Jwts.parser()
                    .setSigningKey(secret)
                    .parseClaimsJws(token)
                    .getBody();
        } catch (Exception e) {
            claims = null;
        }
        return claims;
    }
```
filter部分代码

```
 final String requestHeader = request.getHeader(this.tokenHeader);

        String username = null;
        String authToken = null;
        if (requestHeader != null && requestHeader.startsWith("Bearer ")) {
            authToken = requestHeader.substring(7);
            try {
                username = jwtTokenUtil.getUsernameFromToken(authToken);
            } catch (IllegalArgumentException e) {
                logger.error("an error occured during getting username from token", e);
            } catch (ExpiredJwtException e) {
                logger.warn("the token is expired and not valid anymore", e);
            }
        } else {
            logger.warn("couldn't find bearer string, will ignore the header");
        }

        logger.info("checking authentication for user " + username);
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {

            // It is not compelling necessary to load the use details from the database. You could also store the information
            // in the token and read it from it. It's up to you ;)
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

            // For simple validation it is completely sufficient to just check the token integrity. You don't have to call
            // the database compellingly. Again it's up to you ;)
            if (jwtTokenUtil.validateToken(authToken, userDetails)) {
                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                logger.info("authenticated user " + username + ", setting security context");
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
```

#### Web入口配置

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter{
    
}
```
config中添加filter:
```
        // Custom JWT based security filter
        httpSecurity
                .addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class);
```

#### 核心接口及其实现
- WebSecurityConfig

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter{
    
}
```

- UserDetailsService 需要实现，用来获取用户信息

```
package org.springframework.security.core.userdetails;

public interface UserDetailsService {
    UserDetails loadUserByUsername(String var1) throws UsernameNotFoundException;
}

```

- UserDetails需要实现，关联用户信息

```
package org.springframework.security.core.userdetails;

import java.io.Serializable;
import java.util.Collection;
import org.springframework.security.core.GrantedAuthority;

public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();

    String getPassword();

    String getUsername();

    boolean isAccountNonExpired();

    boolean isAccountNonLocked();

    boolean isCredentialsNonExpired();

    boolean isEnabled();
}
```
#### Demo