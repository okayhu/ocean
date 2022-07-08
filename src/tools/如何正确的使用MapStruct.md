在我们的项目开发过程中，MapStruct 作为一款 pojo 转换工具无疑是十分强大的，便捷的。

但是 MapStruct 虽好，也存在着一些使用误区。

### 不要改变源 pojo 的属性

e.g. 将 UserDTO 对象转换为 UserVO

   ```java
   public class UserDTO {
   
       private Long id;
       private String username;
       private Integer age;
   }
   
   public class UserVO {
   
       private Long id;
       private String username;
       private Integer age;
       private List<RoleVO> roles;
       private List<GroupVO> groups;
       private List<String> authorityCodes;
   }
   
   @Mapper
   public interface UserMapper {
   
       UserVO dtoToVo(UserDTO userDto);
   
       @BeforeMapping
       default void afterDtoToVo(UserDTO userDto) {
           if ("Administrator".equals(userDto.getUsername())) {
               userDto.setId(0L);
               userDto.setAge(null);
           }
       }
   }
   ```

问题：对于方法的调用者来说，会造成预期之外的异常。

### 在 Mapper 中不要包含大量的业务逻辑

   ```java
   @Mapper
   public interface UserMapper {
   
       UserVO dtoToVo(UserDTO userDto);
   
       @AfterMapping
       default void afterDtoToVo(@MappingTarget UserVO userVo) {
           UserService userService = SpringContextUtil.getBean(UserService.class);
           userVo.setAuthorityCodes(userService.getAllAuthorityCodes(userVo.getId()));
       }
   }
   ```
问题：   

   - Mapper 中的方法，很大程度会被多次调用，应用在不同的领域，所以编写者需要确保方法的预期是通用的、可信任的。而在上述写法中，转换后的 UserVO 不包含 roles、groups 包含 authorityCodes，但是这对于方法调用者来说不一定是符合预期的。
   - 一旦修改 Mapper 中的业务逻辑，可能会对多处造成不好的影响。