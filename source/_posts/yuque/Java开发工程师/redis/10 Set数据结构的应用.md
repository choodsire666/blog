**笔记来源：**[**黑马程序员Redis入门到实战教程，深度透析redis底层原理+redis分布式锁+企业解决方案**](https://www.bilibili.com/video/BV1cr4y1671t/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)
# 1 达人探店
## 1.1 发布探店笔记
发布探店笔记<br />探店笔记类似点评网站的评价，往往是图文结合。对应的表有两个：<br />tb_blog：探店笔记表，包含笔记中的标题、文字、图片等
```sql
DROP TABLE IF EXISTS `tb_blog`;
CREATE TABLE `tb_blog`  (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  `shop_id` bigint(20) NOT NULL COMMENT '商户id',
  `user_id` bigint(20) UNSIGNED NOT NULL COMMENT '用户id',
  `title` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '标题',
  `images` varchar(2048) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '探店的照片，最多9张，多张以\",\"隔开',
  `content` varchar(2048) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '探店的文字描述',
  `liked` int(8) UNSIGNED NULL DEFAULT 00000000 COMMENT '点赞数量',
  `comments` int(8) UNSIGNED NULL DEFAULT NULL COMMENT '评论数量',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 8 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
```
tb_blog_comments：其他用户对探店笔记的评价
```sql
DROP TABLE IF EXISTS `tb_blog_comments`;
CREATE TABLE `tb_blog_comments`  (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_id` bigint(20) UNSIGNED NOT NULL COMMENT '用户id',
  `blog_id` bigint(20) UNSIGNED NOT NULL COMMENT '探店id',
  `parent_id` bigint(20) UNSIGNED NOT NULL COMMENT '关联的1级评论id，如果是一级评论，则值为0',
  `answer_id` bigint(20) UNSIGNED NOT NULL COMMENT '回复的评论id',
  `content` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '回复的内容',
  `liked` int(8) UNSIGNED NULL DEFAULT NULL COMMENT '点赞数',
  `status` tinyint(1) UNSIGNED NULL DEFAULT NULL COMMENT '状态，0：正常，1：被举报，2：禁止查看',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
```

**具体发布流程**<br />![1653578992639.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665035922413-ae99bd2f-1860-4e34-9a65-a68b45f040d3.png#averageHue=%23f1edeb&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&height=532&id=u49c34ede&originHeight=680&originWidth=1567&originalType=binary&ratio=1&rotation=0&showTitle=false&size=521262&status=error&style=none&taskId=u6c219c34-65cc-4d1a-8b5e-394b7b67717&title=&width=1225)

上传接口
```java
@Slf4j
@RestController
@RequestMapping("upload")
public class UploadController {

    @PostMapping("blog")
    public Result uploadImage(@RequestParam("file") MultipartFile image) {
        try {
            // 获取原始文件名称
            String originalFilename = image.getOriginalFilename();
            // 生成新文件名
            String fileName = createNewFileName(originalFilename);
            // 保存文件
            image.transferTo(new File(SystemConstants.IMAGE_UPLOAD_DIR, fileName));
            // 返回结果
            log.debug("文件上传成功，{}", fileName);
            return Result.ok(fileName);
        } catch (IOException e) {
            throw new RuntimeException("文件上传失败", e);
        }
    }

}
```
注意：在操作时，需要修改`SystemConstants.IMAGE_UPLOAD_DIR` 自己图片所在的地址，在实际开发中图片一般会放在nginx上或者是云存储上。

BlogController
```java
@RestController
@RequestMapping("/blog")
public class BlogController {

    @Resource
    private IBlogService blogService;

    @PostMapping
    public Result saveBlog(@RequestBody Blog blog) {
        //获取登录用户
        UserDTO user = UserHolder.getUser();
        blog.setUpdateTime(user.getId());
        //保存探店博文
        blogService.saveBlog(blog);
        //返回id
        return Result.ok(blog.getId());
    }
}
```
## 1.2 查看探店笔记
实现查看发布探店笔记的接口

![1653579931626.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665035935991-850bf3e1-0722-4cc8-9369-0a8fe762e270.png#averageHue=%23ece4e1&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=ud672d230&originHeight=462&originWidth=967&originalType=binary&ratio=1&rotation=0&showTitle=false&size=202735&status=error&style=none&taskId=u9d1e6b2a-37d0-4cec-b43c-097408b6f3f&title=)

实现代码：
```java
@Override
public Result queryBlogById(Long id) {
    // 1.查询blog
    Blog blog = getById(id);
    if (blog == null) {
        return Result.fail("笔记不存在！");
    }
    // 2.查询blog有关的用户
    queryBlogUser(blog);
  
    return Result.ok(blog);
}
```
## 1.3 点赞功能
初始代码
```java
@GetMapping("/likes/{id}")
public Result queryBlogLikes(@PathVariable("id") Long id) {
    //修改点赞数量
    blogService.update().setSql("liked = liked +1 ").eq("id",id).update();
    return Result.ok();
}
```
问题分析：这种方式会导致一个用户无限点赞，明显是不合理的<br />造成这个问题的原因是，我们现在的逻辑，发起请求只是给数据库+1，所以才会出现这个问题<br />![1653581590453.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665035949235-bc706076-839d-45df-88ff-0103005d5074.png#averageHue=%23d8d0c8&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&height=555&id=u8fd1ae9d&originHeight=707&originWidth=1573&originalType=binary&ratio=1&rotation=0&showTitle=false&size=934730&status=error&style=none&taskId=u78647854-edfa-411d-af2a-4e845dd0718&title=&width=1235)<br />完善点赞功能<br />需求：

- 同一个用户只能点赞一次，再次点击则取消点赞
- 如果当前用户已经点赞，则点赞按钮高亮显示（前端已实现，判断字段Blog类的isLike属性）

实现步骤：

- 给Blog类中添加一个isLike字段，标示是否被当前用户点赞
- 修改点赞功能，利用Redis的set集合判断是否点赞过，未点赞过则点赞数+1，已点赞过则点赞数-1
- 修改根据id查询Blog的业务，判断当前登录用户是否点赞过，赋值给isLike字段
- 修改分页查询Blog业务，判断当前登录用户是否点赞过，赋值给isLike字段

为什么采用set集合：<br />因为我们的数据是不能重复的，当用户操作过之后，无论他怎么操作，都是

具体步骤：

1. 在Blog 添加一个字段
```java
package com.hmdp.entity;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.Data;
import lombok.EqualsAndHashCode;
import lombok.experimental.Accessors;

import java.io.Serializable;
import java.time.LocalDateTime;


@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@TableName("tb_blog")
public class Blog implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 主键
     */
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;
    /**
     * 商户id
     */
    private Long shopId;
    /**
     * 用户id
     */
    private Long userId;
    /**
     * 用户图标   此字段不会入库
     */
    @TableField(exist = false)
    private String icon;
    /**
     * 用户姓名   此字段不会入库
     */
    @TableField(exist = false)
    private String name;
    /**
     * 是否点赞过了   此字段不会入库，主要是返还给前端，判断点赞按钮是否高亮
     */
    @TableField(exist = false)
    private Boolean isLike;

    /**
     * 标题
     */
    private String title;

    /**
     * 探店的照片，最多9张，多张以","隔开
     */
    private String images;

    /**
     * 探店的文字描述
     */
    private String content;

    /**
     * 点赞数量
     */
    private Integer liked;

    /**
     * 评论数量
     */
    private Integer comments;

    /**
     * 创建时间
     */
    private LocalDateTime createTime;

    /**
     * 更新时间
     */
    private LocalDateTime updateTime;


}

```

2. 修改代码
```java
@Override
public Result likeBlog(Long id){
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
	// 2.判断当前登录用户是否已经点赞
	String key = BLOG_LIKED_KEY + id;
	Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
	if(BooleanUtil.isFalse(isMember)){
    	//3.如果未点赞，可以点赞
    	//3.1 数据库点赞数+1
    	boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
    	//3.2 保存用户到Redis的set集合
    	if(isSuccess){
        	stringRedisTemplate.opsForSet().add(key,userId.toString());
    	}
	}else{
    	//4.如果已点赞，取消点赞
    	//4.1 数据库点赞数-1
    	boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
    	//4.2 把用户从Redis的set集合移除
    	if(isSuccess){
        	stringRedisTemplate.opsForSet().remove(key,userId.toString());
    	}
}
```
## 1.4 点赞排行榜
在探店笔记的详情页面，应该把给该笔记点赞的人显示出来，比如最早点赞的TOP5，形成点赞排行榜<br />之前的点赞是放到set集合，但是set集合是不能排序的，所以这个时候，咱们可以采用一个可以排序的set集合，就是咱们的sortedSet<br />![1653805077118.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665035962213-e972ab55-1363-43ea-9ccb-fcf744cf5d46.png#averageHue=%23e4dddc&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=ufd783a37&originHeight=633&originWidth=1306&originalType=binary&ratio=1&rotation=0&showTitle=false&size=461034&status=error&style=none&taskId=udb6d57a5-42c2-4027-b538-fad6bbdc6e5&title=)

我们接下来来对比一下这些集合的区别是什么<br />所有点赞的人，需要是唯一的，所以我们应当使用set或者是sortedSet<br />其次我们需要排序，就可以直接锁定使用sortedSet啦<br />![1653805203758.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665035974909-417b24d9-2c19-472b-b9ec-96cf677fa455.png#averageHue=%23c9c9c6&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=udf96c1dd&originHeight=412&originWidth=1167&originalType=binary&ratio=1&rotation=0&showTitle=false&size=121498&status=error&style=none&taskId=ub04f9c8e-8a9d-4c2f-a457-1b75d4d3440&title=)<br />修改代码<br />点赞逻辑代码
```java
@Override
public Result likeBlog(Long id) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.判断当前登录用户是否已经点赞
    String key = BLOG_LIKED_KEY + id;
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    if (score == null) {
        // 3.如果未点赞，可以点赞
        // 3.1.数据库点赞数 + 1
        boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
        // 3.2.保存用户到Redis的set集合  zadd key value score 此处是按照时间戳来进行排序的
        if (isSuccess) {
            stringRedisTemplate.opsForZSet().add(key, userId.toString(), System.currentTimeMillis());
        }
    } else {
        // 4.如果已点赞，取消点赞
        // 4.1.数据库点赞数 -1
        boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
        // 4.2.把用户从Redis的set集合移除
        if (isSuccess) {
            stringRedisTemplate.opsForZSet().remove(key, userId.toString());
        }
    }
    return Result.ok();
}


private void isBlogLiked(Blog blog) {
    // 1.获取登录用户
    UserDTO user = UserHolder.getUser();
    if (user == null) {
        // 用户未登录，无需查询是否点赞
        return;
    }
    Long userId = user.getId();
    // 2.判断当前登录用户是否已经点赞
    String key = "blog:liked:" + blog.getId();
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    blog.setIsLike(score != null);
}
```
 <br />点赞列表查询列表
```java
@GetMapping("/likes/{id}")
public Result queryBlogLikes(@PathVariable("id") Long id) {
    return blogService.queryBlogLikes(id);
}
```
```java
@Override
public Result queryBlogLikes(Long id) {
    String key = BLOG_LIKED_KEY + id;
    // 1.查询top5的点赞用户 zrange key 0 4
    Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
    if (top5 == null || top5.isEmpty()) {
        return Result.ok(Collections.emptyList());
    }
    // 2.解析出其中的用户id
    List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
    String idStr = StrUtil.join(",", ids);
    // 3.根据用户id查询用户 WHERE id IN ( 5 , 1 ) ORDER BY FIELD(id, 5, 1)
    List<UserDTO> userDTOS = userService.query()
            .in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list()
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    // 4.返回
    return Result.ok(userDTOS);
}
```

# 2 好友关注
## 2.1 关注和取消关注
针对用户的操作：可以对用户进行关注和取消关注功能。<br />![1653806140822.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665035991717-bc923891-090b-49fe-b912-3e7b554c4410.png#averageHue=%23e4dfda&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&height=509&id=ue412bf00&originHeight=694&originWidth=1404&originalType=binary&ratio=1&rotation=0&showTitle=false&size=554537&status=error&style=none&taskId=ue176ba3f-52f0-41a9-be24-8b0e4b33bf1&title=&width=1030)

实现思路：<br />需求：基于该表数据结构，实现两个接口：

- 关注和取关接口
- 判断是否关注的接口

关注是User之间的关系，是博主与粉丝的关系，数据库中有一张tb_follow表来标示：
```plsql
DROP TABLE IF EXISTS `tb_follow`;
CREATE TABLE `tb_follow`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_id` bigint(20) UNSIGNED NOT NULL COMMENT '用户id',
  `follow_user_id` bigint(20) UNSIGNED NOT NULL COMMENT '关联的用户id',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;

```
注意: 这里需要把主键修改为自增长，简化开发。<br />FollowController
```java
//关注或者取消关注
@PutMapping("/{id}/{isFollow}")
public Result follow(@PathVariable("id") Long followUserId, @PathVariable("isFollow") Boolean isFollow) {
    return followService.follow(followUserId, isFollow);
}
//查看是否关注该用户
@GetMapping("/or/not/{id}")
public Result isFollow(@PathVariable("id") Long followUserId) {
      return followService.isFollow(followUserId);
}
```

FollowService
```java
//是否关注该用户
@Override
public Result isFollow(Long followUserId) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.查询是否关注 select count(*) from tb_follow where user_id = ? and follow_user_id = ?
    Integer count = query().eq("user_id", userId).eq("follow_user_id", followUserId).count();
    // 3.判断
    return Result.ok(count > 0);
}

//关注或者取消关注service
@Override
public Result follow(Long followUserId, Boolean isFollow) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    // 1.判断到底是关注还是取关
    if (isFollow) {
        // 2.关注，新增数据
        Follow follow = new Follow();
        follow.setUserId(userId);
        follow.setFollowUserId(followUserId);
        boolean isSuccess = save(follow);

    } else {
        // 3.取关，删除 delete from tb_follow where user_id = ? and follow_user_id = ?
        remove(new QueryWrapper<Follow>()
                    .eq("user_id", userId).eq("follow_user_id", followUserId));

    }
    return Result.ok();
}
```

## 2.2 共同关注
想要去看共同关注的好友，需要首先进入到这个页面，这个页面会发起两个请求

1. 去查询用户的详情
2. 去查询用户的笔记

以上两个功能和共同关注没有什么关系，大家可以自行将笔记中的代码拷贝到idea中就可以实现这两个功能了，我们的重点在于共同关注功能。<br />![1653806706296.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036024843-a0b2cf6d-c9c0-4bd3-8b62-c11dafddc612.png#averageHue=%23e2dcd7&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=uc587ba5f&originHeight=630&originWidth=1602&originalType=binary&ratio=1&rotation=0&showTitle=false&size=636476&status=error&style=none&taskId=u21ea32e6-5959-4e83-a1af-44980b9ba4a&title=)
```java
// UserController 根据id查询用户
@GetMapping("/{id}")
public Result queryUserById(@PathVariable("id") Long userId){
	// 查询详情
	User user = userService.getById(userId);
	if (user == null) {
		return Result.ok();
	}
	UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
	// 返回
	return Result.ok(userDTO);
}

// BlogController  根据id查询博主的探店笔记
@GetMapping("/of/user")
public Result queryBlogByUserId(
		@RequestParam(value = "current", defaultValue = "1") Integer current,
		@RequestParam("id") Long id) {
	// 根据用户查询
	Page<Blog> page = blogService.query()
			.eq("user_id", id).page(new Page<>(current, SystemConstants.MAX_PAGE_SIZE));
	// 获取当前页数据
	List<Blog> records = page.getRecords();
	return Result.ok(records);
}
```
接下来我们来看看共同关注如何实现：<br />需求：利用Redis中恰当的数据结构，实现共同关注功能。在博主个人页面展示出当前用户与博主的共同关注呢。<br />当然是使用我们之前学习过的set集合咯，在set集合中，有交集并集补集的api，我们可以把两人的关注的人分别放入到一个set集合中，然后再通过api去查看这两个set集合中的交集数据。<br />![1653806973212.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036038856-d3bdec1e-baee-4ffd-90e8-0ab29135c3cd.png#averageHue=%23fcfbfb&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=ub918d64d&originHeight=563&originWidth=1259&originalType=binary&ratio=1&rotation=0&showTitle=false&size=108933&status=error&style=none&taskId=u2341defd-bda6-480a-ba42-6102738f452&title=)

我们先来改造当前的关注列表<br />改造原因是因为我们需要在用户关注了某位用户后，需要将数据放入到set集合中，方便后续进行共同关注，同时当取消关注时，也需要从set集合中进行删除<br />FollowServiceImpl
```java
@Override
public Result follow(Long followUserId, Boolean isFollow) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    // 1.判断到底是关注还是取关
    if (isFollow) {
        // 2.关注，新增数据
        Follow follow = new Follow();
        follow.setUserId(userId);
        follow.setFollowUserId(followUserId);
        boolean isSuccess = save(follow);
        if (isSuccess) {
            // 把关注用户的id，放入redis的set集合 sadd userId followerUserId
            stringRedisTemplate.opsForSet().add(key, followUserId.toString());
        }
    } else {
        // 3.取关，删除 delete from tb_follow where user_id = ? and follow_user_id = ?
        boolean isSuccess = remove(new QueryWrapper<Follow>()
                .eq("user_id", userId).eq("follow_user_id", followUserId));
        if (isSuccess) {
            // 把关注用户的id从Redis集合中移除
            stringRedisTemplate.opsForSet().remove(key, followUserId.toString());
        }
    }
    return Result.ok();
}
```

**具体的公用关注代码：**
```java
@Override
public Result followCommons(Long id) {
    // 1.获取当前用户
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    // 2.求交集
    String key2 = "follows:" + id;
    Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key, key2);
    if (intersect == null || intersect.isEmpty()) {
        // 无交集
        return Result.ok(Collections.emptyList());
    }
    // 3.解析id集合
    List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
    // 4.查询用户
    List<UserDTO> users = userService.listByIds(ids)
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    return Result.ok(users);
}
```

## 2.3 Feed流实现方案
当我们关注了用户后，这个用户发了动态，那么我们应该把这些数据推送给用户，这个需求，其实我们又把他叫做Feed流，关注推送也叫做Feed流，直译为投喂。为用户持续的提供“沉浸式”的体验，通过无限下拉刷新获取新的信息。<br />对于传统的模式的内容解锁：我们是需要用户去通过搜索引擎或者是其他的方式去解锁想要看的内容<br />![1653808641260.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036058879-32984625-05a6-4454-964a-2f56f35725a4.png#averageHue=%23e6d6d6&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=ufc0ebe56&originHeight=175&originWidth=1060&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19348&status=error&style=none&taskId=u2feee938-e8fd-4378-aa60-889fba57664&title=)<br />对于新型的Feed流的的效果：不需要我们用户再去推送信息，而是系统分析用户到底想要什么，然后直接把内容推送给用户，从而使用户能够更加的节约时间，不用主动去寻找。<br />![1653808993693.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036068281-96f181b6-9f8f-4c81-881f-3d7aa47f28af.png#averageHue=%23e8d9d9&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=uf47630df&originHeight=188&originWidth=1062&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19490&status=error&style=none&taskId=u4d8b625c-5d5b-44c7-8d32-6e1eb80fcaf&title=)<br />Feed流产品有两种常见模式：<br />**Timeline**：不做内容筛选，简单的按照内容发布时间排序，常用于好友或关注。例如朋友圈

- 优点：信息全面，不会有缺失。并且实现也相对简单
- 缺点：信息噪音较多，用户不一定感兴趣，内容获取效率低

**智能排序**：利用智能算法屏蔽掉违规的、用户不感兴趣的内容。推送用户感兴趣信息来吸引用户

- 优点：投喂用户感兴趣信息，用户粘度很高，容易沉迷
- 缺点：如果算法不精准，可能起到反作用

本例中的个人页面，是基于关注的好友来做Feed流，因此采用Timeline的模式。该模式的实现方案有三种：<br />我们本次针对好友的操作，采用的就是Timeline的方式，只需要拿到我们关注用户的信息，然后按照时间排序即可，因此采用Timeline的模式。该模式的实现方案有三种：

- 拉模式
- 推模式
- 推拉结合

**拉模式**：也叫做读扩散<br />该模式的核心含义就是：当张三和李四和王五发了消息后，都会保存在自己的邮箱中，假设赵六要读取信息，那么他会从读取他自己的收件箱，此时系统会从他关注的人群中，把他关注人的信息全部都进行拉取，然后在进行排序<br />优点：比较节约空间，因为赵六在读信息时，并没有重复读取，而且读取完之后可以把他的收件箱进行清楚。<br />缺点：比较延迟，当用户读取数据时才去关注的人里边去读取数据，假设用户关注了大量的用户，那么此时就会拉取海量的内容，对服务器压力巨大。

![1653809450816.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036082867-db1edb7a-6096-41a9-a62e-e0dbe3e346ac.png#averageHue=%23fcfcfb&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=u9833404d&originHeight=637&originWidth=1474&originalType=binary&ratio=1&rotation=0&showTitle=false&size=204719&status=error&style=none&taskId=u831d901a-8c97-4bb0-bd16-421adf432fe&title=)

**推模式**：也叫做写扩散。<br />推模式是没有写邮箱的，当张三写了一个内容，此时会主动的把张三写的内容发送到他的粉丝收件箱中去，假设此时李四再来读取，就不用再去临时拉取了<br />优点：时效快，不用临时拉取<br />缺点：内存压力大，假设一个大V写信息，很多人关注他， 就会写很多分数据到粉丝那边去<br />![1653809875208.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036092253-524f77d9-996f-4aad-a37e-253876a324b0.png#averageHue=%23fcfcfc&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=u4fd49c07&originHeight=688&originWidth=1012&originalType=binary&ratio=1&rotation=0&showTitle=false&size=162929&status=error&style=none&taskId=u74b15c9d-84d0-4c26-9a7f-98af35633c2&title=)<br />**推拉结合模式**：也叫做读写混合，兼具推和拉两种模式的优点。<br />推拉模式是一个折中的方案，站在发件人这一段，如果是个普通的人，那么我们采用写扩散的方式，直接把数据写入到他的粉丝中去，因为普通的人他的粉丝关注量比较小，所以这样做没有压力，如果是大V，那么他是直接将数据先写入到一份到发件箱里边去，然后再直接写一份到活跃粉丝收件箱里边去，现在站在收件人这端来看，如果是活跃粉丝，那么大V和普通的人发的都会直接写入到自己收件箱里边来，而如果是普通的粉丝，由于他们上线不是很频繁，所以等他们上线时，再从发件箱里边去拉信息。<br />![1653812346852.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036102647-a51d05aa-1ace-4783-853c-22dee8f996a0.png#averageHue=%23fcfcfb&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=u9455c3ce&originHeight=640&originWidth=1442&originalType=binary&ratio=1&rotation=0&showTitle=false&size=252025&status=error&style=none&taskId=udd366193-3e97-46ed-bdfc-9227d3a823f&title=)
## 2.4 推送到粉丝收件箱
需求：

- 修改新增探店笔记的业务，在保存blog到数据库的同时，推送到粉丝的收件箱
- 收件箱满足可以根据时间戳排序，必须用Redis的数据结构实现
- 查询收件箱数据时，可以实现分页查询

Feed流中的数据会不断更新，所以数据的角标也在变化，因此不能采用传统的分页模式。<br />传统了分页在feed流是不适用的，因为我们的数据会随时发生变化<br />假设在t1 时刻，我们去读取第一页，此时page = 1 ，size = 5 ，那么我们拿到的就是10~6 这几条记录，假设现在t2时候又发布了一条记录，此时t3 时刻，我们来读取第二页，读取第二页传入的参数是page=2 ，size=5 ，那么此时读取到的第二页实际上是从6 开始，然后是6~2 ，那么我们就读取到了重复的数据，所以feed流的分页，不能采用原始方案来做。<br />![1653813047671.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036115069-50504856-660f-4f17-82d6-6fa94806a61a.png#averageHue=%23f9f8f7&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=uc4b995eb&originHeight=634&originWidth=1431&originalType=binary&ratio=1&rotation=0&showTitle=false&size=154550&status=error&style=none&taskId=u6b1c0955-5988-4e28-8256-96a934c4319&title=)


Feed流的滚动分页<br />我们需要记录每次操作的最后一条，然后从这个位置开始去读取数据<br />举个例子：我们从t1时刻开始，拿第一页数据，拿到了10~6，然后记录下当前最后一次拿取的记录，就是6，t2时刻发布了新的记录，此时这个11放到最顶上，但是不会影响我们之前记录的6，此时t3时刻来拿第二页，第二页这个时候拿数据，还是从6后一点的5去拿，就拿到了5-1的记录。我们这个地方可以采用sortedSet来做，可以进行范围查询，并且还可以记录当前获取数据时间戳最小值，就可以实现滚动分页了

![1653813462834.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036168603-937f7173-8d87-484c-8a4b-97020a1f3a53.png#averageHue=%23f8f7f7&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=u1d53351a&originHeight=675&originWidth=1289&originalType=binary&ratio=1&rotation=0&showTitle=false&size=152549&status=error&style=none&taskId=u23dd940d-9ca8-43dc-b257-46794f6d62b&title=)<br />核心的意思：就是我们在保存完探店笔记后，获得到当前笔记的粉丝，然后把数据推送到粉丝的redis中去。
```java
@Override
public Result saveBlog(Blog blog) {
    // 1.获取登录用户
    UserDTO user = UserHolder.getUser();
    blog.setUserId(user.getId());
    // 2.保存探店笔记
    boolean isSuccess = save(blog);
    if(!isSuccess){
        return Result.fail("新增笔记失败!");
    }
    // 3.查询笔记作者的所有粉丝 select * from tb_follow where follow_user_id = ?
    List<Follow> follows = followService.query().eq("follow_user_id", user.getId()).list();
    // 4.推送笔记id给所有粉丝
    for (Follow follow : follows) {
        // 4.1.获取粉丝id
        Long userId = follow.getUserId();
        // 4.2.推送
        String key = FEED_KEY + userId;
        stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
    }
    // 5.返回id
    return Result.ok(blog.getId());
}
```
## 2.5 实现分页查询收邮箱
需求：在个人主页的“关注”卡片中，查询并展示推送的Blog信息：<br />具体操作如下：

1. 每次查询完成后，我们要分析出查询出数据的最小时间戳，这个值会作为下一次查询的条件
2. 我们需要找到与上一次查询相同的查询个数作为偏移量，下次查询时，跳过这些查询过的数据，拿到我们需要的数据

综上：我们的请求参数中就需要携带 lastId：上一次查询的最小时间戳 和偏移量这两个参数。<br />这两个参数第一次会由前端来指定，以后的查询就根据后台结果作为条件，再次传递到后台。

![1653819821591.png](https://cdn.nlark.com/yuque/0/2022/png/22334924/1665036183018-3cbc6572-fc0c-431e-bdeb-f340f7619e47.png#averageHue=%23dfd7d4&clientId=u94722be3-b773-4&errorMessage=unknown%20error&from=drop&id=u2be53cf2&originHeight=624&originWidth=1244&originalType=binary&ratio=1&rotation=0&showTitle=false&size=408608&status=error&style=none&taskId=u302e7b21-eddc-4534-8dbe-b0120e3c2f6&title=)

定义出来具体的返回值实体类
```java
@Data
public class ScrollResult {
    private List<?> list;
    private Long minTime;
    private Integer offset;
}
```

BlogController<br />注意：RequestParam 表示接受url地址栏传参的注解，当方法上参数的名称和url地址栏不相同时，可以通过RequestParam 来进行指定
```java
@GetMapping("/of/follow")
public Result queryBlogOfFollow(
    @RequestParam("lastId") Long max, @RequestParam(value = "offset", defaultValue = "0") Integer offset){
    return blogService.queryBlogOfFollow(max, offset);
}
```

BlogServiceImpl
```java
@Override
public Result queryBlogOfFollow(Long max, Integer offset) {
    // 1.获取当前用户
    Long userId = UserHolder.getUser().getId();
    // 2.查询收件箱 ZREVRANGEBYSCORE key Max Min LIMIT offset count
    String key = FEED_KEY + userId;
    Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet()
        .reverseRangeByScoreWithScores(key, 0, max, offset, 2);
    // 3.非空判断
    if (typedTuples == null || typedTuples.isEmpty()) {
        return Result.ok();
    }
    // 4.解析数据：blogId、minTime（时间戳）、offset
    List<Long> ids = new ArrayList<>(typedTuples.size());
    long minTime = 0; // 2
    int os = 1; // 2
    for (ZSetOperations.TypedTuple<String> tuple : typedTuples) { // 5 4 4 2 2
        // 4.1.获取id
        ids.add(Long.valueOf(tuple.getValue()));
        // 4.2.获取分数(时间戳）
        long time = tuple.getScore().longValue();
        if(time == minTime){
            os++;
        }else{
            minTime = time;
            os = 1;
        }
    }
	os = minTime == max ? os : os + offset;
    // 5.根据id查询blog
    String idStr = StrUtil.join(",", ids);
    List<Blog> blogs = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();

    for (Blog blog : blogs) {
        // 5.1.查询blog有关的用户
        queryBlogUser(blog);
        // 5.2.查询blog是否被点赞
        isBlogLiked(blog);
    }

    // 6.封装并返回
    ScrollResult r = new ScrollResult();
    r.setList(blogs);
    r.setOffset(os);
    r.setMinTime(minTime);

    return Result.ok(r);
}
```

