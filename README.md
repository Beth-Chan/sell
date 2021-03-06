# 基于Spring Boot的外卖系统

### 项目编写过程记录

> - 数据库设计
> - 日志编写
> - 商品类目ProductCategory DAO层设计与开发
> - 商品类目ProductCategory Service层设计与开发
> - 商品信息ProductInfo 相关操作(包括DAO、Service、Controller和VO)
> - 订单详情OrderDetail和主订单OrderMaster相关操作（主要看Service层）

#### 数据库设计
(采用Navicat for MySQL操作，可以新建查询，或者可视化操作）
```
CREATE TABLE `product_info` (
    `product_id` VARCHAR(32) NOT NULL,
    `product_name` VARCHAR(64) NOT NULL,
    `product_price` DECIMAL(8, 2) NOT NULL,
    `product_stock` INT NOT NULL,
    `product_description` VARCHAR(64),
    `product_icon` VARCHAR(512),
    `product_status` INT NOT NULL,
    `category_type` INT NOT NULL,
    `create_time` TIMESTAMP NOT NULL DEFAULT current_timestamp,
    `update_time` TIMESTAMP  NOT NULL DEFAULT current_timestamp ON UPDATE current_timestamp
);

CREATE TABLE `product_category` (
    `category_id` INT NOT NULL AUTO_INCREMENT,
    `category_name` VARCHAR(64) NOT NULL,
    `category_type` INT NOT NULL,
    `create_time` TIMESTAMP NOT NULL DEFAULT current_timestamp,
    `update_time` TIMESTAMP  NOT NULL DEFAULT current_timestamp ON UPDATE current_timestamp,
    PRIMARY KEY(`category_id`),
    UNIQUE KEY unique_category_type(`category_type`)
);

CREATE TABLE `order_master` (
    `order_id` VARCHAR(32) NOT NULL,
    `buyer_name` VARCHAR(32) NOT NULL,
    `buyer_phone` VARCHAR(32) NOT NULL,
    `buyer_address` VARCHAR(128) NOT NULL,
    `buyer_openid` VARCHAR(32) COMMENT '买家微信id',
    `order_amount` DECIMAL(8, 2) NOT NULL,
    `order_status` TINYINT(3) NOT NULL DEFAULT '0' COMMENT '默认0新下单',
    `pay_status` TINYINT(3) NOT NULL DEFAULT '0' COMMENT '默认0是未支付',
    `create_time` TIMESTAMP NOT NULL DEFAULT current_timestamp,
    `update_time` TIMESTAMP  NOT NULL DEFAULT current_timestamp ON UPDATE current_timestamp,
    PRIMARY KEY(`order_id`),
    KEY `idx_buyer_openid` (`buyer_openid`)
);

CREATE TABLE `order_detail` (
    `detail_id` VARCHAR(32) NOT NULL,
    `order_id` VARCHAR(32) NOT NULL,
    `product_id` VARCHAR(32) NOT NULL,
    `product_name` VARCHAR(64) NOT NULL,
    `product_price` DECIMAL(8, 2) NOT NULL,
    `product_quantity` INT NOT NULL,
    `product_icon` VARCHAR(512),
    `create_time` TIMESTAMP NOT NULL DEFAULT current_timestamp,
    `update_time` TIMESTAMP  NOT NULL DEFAULT current_timestamp ON UPDATE current_timestamp,
    PRIMARY KEY(`detail_id`),
    KEY `idx_order_id` (`order_id`)
);
```

#### 日志编写

pom.xml里添加依赖：    
```
<!--此插件支持日志和getter，setter和toString-->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

新建resources/logback-spring.xml：
```
<?xml version="1.0" encoding="UTF-8" ?>
<configuration>
    <!--配置输出控制台日志-->
    <appender name="consoleLog" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>
                %d - %msg%n
            </pattern>
        </layout>
    </appender>

    <!--配置输出文件日志，分为Info文件和Error文件-->
    <appender name="fileInfoLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>DENY</onMatch>
            <onMismatch>ACCEPT</onMismatch>
        </filter>
        <encoder>
            <pattern>
                %msg%n
            </pattern>
        </encoder>
        <!--滚动策略-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>var/log/tomcat/sell/info.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <appender name="fileErrorLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <encoder>
            <pattern>
                %msg%n
            </pattern>
        </encoder>
        <!--滚动策略-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>var/log/tomcat/sell/error.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <root level="info">
        <appender-ref ref="consoleLog" />
        <appender-ref ref="fileInfoLog" />
        <appender-ref ref="fileErrorLog" />
    </root>
</configuration>
```


#### 商品类目ProductCategory**DAO**层设计与开发

**第一个表product_category的操作分五步：**

第一步.pom.xml里添加依赖
```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```


第二步.添加数据库相关配置，修改application.properties为application.yml，树形结构更简洁：
```
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: ********(此处要改成数据库对应的用户名）
    password: ********(此处要改成对应的密码）
    url: jdbc:mysql://123.207.95.134/sell?characterEncoding=utf-8&useSSL=false
  jpa:
    show-sql: true
```


第三步.创建dataobject文件夹存放Entity实体对象，创建数据库对象（product_category表的映射）<br>
ProductCategory类：<br>
```
@Entity
@DynamicUpdate
@Data /* lombok自动生成getter, setter and toString() */
public class ProductCategory {
    /** 类目id. */
    @Id /* 主键 */
    @GeneratedValue /* 自增 */
    private Integer categoryId;

    private String categoryName;

    private Integer categoryType;

    public ProductCategory() {
    }

    public ProductCategory(String categoryName, Integer categoryType) {
        this.categoryName = categoryName;
        this.categoryType = categoryType;
    }
}
```
@Entity表示当前类是实体类；@Id表示当前属性是主键；@GeneratedValue表示自增


第四步.创建repository文件夹存放DAO Bean，写DAO层的代码（直接extends JpaRepository，连SQL语句都不用写）<br>
ProductCategoryRepository是一个接口：
```
// ProductCategory和Integer是对象和主键类型
public interface ProductCategoryRepository extends JpaRepository<ProductCategory, Integer> {

    // 通过category_type查商品列表
    List<ProductCategory> findByCategoryTypeIn(List<Integer> categoryTypeList);
}
```


第五步.进行单元测试
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class ProductCategoryRepositoryTest {

   @Autowired
   private ProductCategoryRepository repository;

   @Test
   public void findOneTest() {
       // 找出id为1的记录
       ProductCategory productCategory = repository.findOne(1);
       System.out.println(productCategory.toString());
   }

   @Test
   @Transactional /* 测试完数据库不要插入数据 */
   public void saveTest() { // 新增或修改的话都是用saveTest
       // 更新往往是先查出来，再判断权限等信息，再可以更改
       // ProductCategory productCategory = repository.findOne(2);
       // productCategory.setCategoryType(10);
       // repository.save(productCategory);

       // // 新增得先构造对象
       // ProductCategory productCategory = new ProductCategory();
       // // 修改要加setCategoryId，新增可以不用，因为默认自增id
       // productCategory.setCategoryId(2);
       // productCategory.setCategoryName("精选热菜");
       // productCategory.setCategoryType(3);
       // repository.save(productCategory);

       ProductCategory productCategory = new ProductCategory("女生最爱", 5);
       ProductCategory result = repository.save(productCategory);
       Assert.assertNotNull(result);
       // 等价于Assert.assertNotEquals(null, result);
   }

   @Test
   public void findByCategoryTypeInTest() {
       List<Integer> list = Arrays.asList(2, 3, 4);
       List<ProductCategory> result = repository.findByCategoryTypeIn(list);
       Assert.assertNotEquals(0, result.size());
   }
}
```

#### 买家类目Service层设计与开发
创建service文件夹，编写CategoryService接口
```
public interface CategoryService {
    /* 后台管理 */
    ProductCategory findOne(Integer categoryId);
    List<ProductCategory> findAll();

    /* 买家端 */
    List<ProductCategory> findByCategoryTypeIn(List<Integer> categoryTypeList);
    // 新增和更新，删除暂时用不到
    ProductCategory save(ProductCategory productCategory);
}
```

写实现类：
```
@Service
public class CategoryServiceImpl implements CategoryService {

    // 引入DAO
    @Autowired
    private ProductCategoryRepository repository;

    @Override
    public ProductCategory findOne(Integer categoryId) {
        return repository.findOne(categoryId);
    }

    @Override
    public List<ProductCategory> findAll() {
        return repository.findAll();
    }

    @Override
    public List<ProductCategory> findByCategoryTypeIn(List<Integer> categoryTypeList) {
        return repository.findByCategoryTypeIn(categoryTypeList);
    }

    @Override
    public ProductCategory save(ProductCategory productCategory) {
        return repository.save(productCategory);
    }
}
```


写测试：
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class CategoryServiceImplTest {

    @Autowired
    CategoryServiceImpl categoryService;

    @Test
    public void findOne() {
        ProductCategory productCategory = categoryService.findOne(1);
        Assert.assertEquals(new Integer(1), productCategory.getCategoryId());
    }

    @Test
    public void findAll() {
        List<ProductCategory> productCategoryList = categoryService.findAll();
        Assert.assertNotEquals(0, productCategoryList.size());
    }

    @Test
    public void findByCategoryTypeIn() {
        List<ProductCategory> productCategoryList = categoryService.findByCategoryTypeIn(Arrays.asList(1, 2, 3, 4));
        Assert.assertNotEquals(0, productCategoryList.size());
    }

    @Test
    public void save() {
        ProductCategory productCategory = new ProductCategory("热销榜", 1);
        ProductCategory result = categoryService.save(productCategory);
        Assert.assertNotNull(result);
    }
}
```
     
     
#### 商品信息ProductInfo相关操作(包括DAO、Service、Controller和VO)

第一步：写ProductInfo实体类
```
@Entity
@Data
public class ProductInfo {
    @Id
    private String productId;
    private String productName;
    // 数据库是decimal(8,2)，要对应BigDecimal
    private BigDecimal productPrice;
    private Integer productStock;
    private String productDescription;
    private String productIcon;
    private Integer productStatus;
    private Integer categoryType;
}
```

第二步：写DAO接口
```
public interface ProductInfoRepository extends JpaRepository<ProductInfo, String> {
    List<ProductInfo> findByProductStatus(Integer productStatus);
}
```

第三步：写DAO层测试
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class ProductInfoRepositoryTest {

    @Autowired
    private ProductInfoRepository repository;

    @Test
    public void saveTest() {
        ProductInfo productInfo = new ProductInfo();
        productInfo.setProductId("1234");
        productInfo.setProductName("皮蛋瘦肉粥");
        productInfo.setProductPrice(new BigDecimal(10));
        productInfo.setProductStock(30);
        productInfo.setProductDescription("很好喝的粥");
        productInfo.setProductIcon("http://xxxx.jpg");
        productInfo.setProductStatus(ProductStatusEnum.UP.getCode());
        productInfo.setCategoryType(3);

        ProductInfo result = repository.save(productInfo);
        Assert.assertNotNull(result);
    }

    @Test
    public void findByProductStatus(){
        List<ProductInfo>  productInfoList = repository.findByProductStatus(0);
        Assert.assertNotEquals(0, productInfoList.size());
    }
}
```

第四步：写Service接口
```
public interface ProductService {
    ProductInfo findOne(String productId);

    // 买家查询所有在架商品列表
    List<ProductInfo> findUpAll();
    // 后台管理可以查看所有商品列表
    Page<ProductInfo> findAll(Pageable pageable);

    ProductInfo save(ProductInfo productInfo);

    // 增加库存

    // 减少库存
}
```

第五步：写Service实现类
```
@Service
public class ProductServiceImpl implements ProductService {

    @Autowired
    private ProductInfoRepository repository;

    @Override
    public ProductInfo findOne(String productId) {
        return repository.findOne(productId);
    }

    @Override
    public List<ProductInfo> findUpAll() {
        return repository.findByProductStatus(ProductStatusEnum.UP.getCode());
    }

    @Override
    public Page<ProductInfo> findAll(Pageable pageable) {
        return repository.findAll(pageable);
    }

    @Override
    public ProductInfo save(ProductInfo productInfo) {
        return repository.save(productInfo);
    }
}
```

第六步：编写Service层测试

``` 
@RunWith(SpringRunner.class)
@SpringBootTest
public class ProductServiceImplTest {

    @Autowired
    private ProductServiceImpl productService;

    @Test
    public void findOne() {
        ProductInfo productInfo = productService.findOne("1234");
        Assert.assertEquals("1234", productInfo.getProductId());
    }

    @Test
    public void findUpAll() {
        List<ProductInfo> productInfoList = (List<ProductInfo>) productService.findUpAll();
        Assert.assertNotEquals(0, productInfoList.size());
    }

    @Test
    public void findAll() {
        // PageRequest继承自AbstractPageRequest，这个类又实现了Pageable接口，Pageable是一个接口，不是具体实现类
        PageRequest request = new PageRequest(0,2);
        Page<ProductInfo> productInfoPage = productService.findAll(request);
        System.out.println(productInfoPage.getTotalElements());
    }

    @Test
    public void save() {
        ProductInfo productInfo = new ProductInfo();
        productInfo.setProductId("12345678");
        productInfo.setProductName("皮皮虾");
        productInfo.setProductPrice(new BigDecimal(68.5));
        productInfo.setProductStock(120);
        productInfo.setProductDescription("很好吃的虾");
        productInfo.setProductIcon("http://xxxxxxxxxxxxxxxxxxxxxxxxxx.jpg");
        productInfo.setProductStatus(ProductStatusEnum.DOWN.getCode());
        productInfo.setCategoryType(2);

        ProductInfo result = productService.save(productInfo);
        Assert.assertNotNull(result);
    }
}
```


第七步：编写与前端交互的VO
```
@Data
public class ResultVO<T> {
    // code等于0代表成功
    private Integer code;
    private String msg;
    private T data;
}
```


第八步：编写Controller
```
@RestController /* 返回json格式 */
@RequestMapping("/buyer/product")
public class BuyerProductController {
    @GetMapping("/list")
    public ResultVO list() {
        ResultVO resultVO = new ResultVO();
        return resultVO;
    }
}
```

至此显示结果是：<br>
![json](./screenshot/json1.jpg))


第九步：根据API接口数据的结构，编写data里的ProductVO和ProductInfoVO对象
ProductVO:
```
@Data
public class ProductVO {

    @JsonProperty("name") /* 返回给前端是name */
    private String categoryName;

    @JsonProperty("type")
    private Integer categoryType;

    @JsonProperty("foods")
    private List<ProductInfoVO> productInfoVOList;
}
```

ProductInfoVO:
```
@Data
public class ProductInfoVO {
    @JsonProperty("id")
    private String productId;
    @JsonProperty("name")
    private String productName;
    @JsonProperty("price")
    private BigDecimal productPrice;
    @JsonProperty("description")
    private String productDescription;
    @JsonProperty("icon")
    private String productIcon;
}
```

修改BuyerProductController:
```
@RestController /* 返回json格式 */
@RequestMapping("/buyer/product")
public class BuyerProductController {
    @GetMapping("/list")
    public ResultVO list() {
        ResultVO resultVO = new ResultVO();
        resultVO.setCode(0);
        resultVO.setMsg("成功");
        ProductVO productVO = new ProductVO();
        resultVO.setData(productVO);
        return resultVO;
    }
}
```

至此得到的结果是:<br>
![json](./screenshot/json2.jpg)

封装工具：
```
public class ResultVOUtils {
    public static ResultVO success(Object object) {
        ResultVO resultVO = new ResultVO();
        resultVO.setData(object);
        resultVO.setCode(0);
        resultVO.setMsg("成功");
        return resultVO;
    }

    public static ResultVO success() {
        return success(null);
    }

    public static ResultVO error(Integer code, String msg) {
        ResultVO resultVO = new ResultVO();
        resultVO.setCode(code);
        resultVO.setMsg(msg);
        return resultVO;
    }
}
```


修改BuyerProductController.java:
```
@RestController /* 返回json格式 */
@RequestMapping("/buyer/product")
public class BuyerProductController {

    @Autowired
    private ProductService productService; /* 查询商品要用到ProductService */

    @Autowired
    private CategoryService categoryService;

    @GetMapping("/list")
    public ResultVO list() {
        // 1.从数据库中查询所有的上架商品
        List<ProductInfo> productInfoList =  productService.findUpAll();

        // 2.从数据库中查询商品类目
        // List<Integer> categoryTypeList = new ArrayList<>();
        // 传统方法
        // 遍历productInfoList，把每一个ProductInfo取出来
        // for(ProductInfo productInfo : productInfoList) {
        //     categoryTypeList.add(productInfo.getCategoryType());
        // }
        // 精简做法（java 8 lambda）
        List<Integer> categoryTypeList =  productInfoList.stream()
                .map(productInfo -> productInfo.getCategoryType())
                .collect(Collectors.toList());
        List<ProductCategory> productCategoryList = categoryService.findByCategoryTypeIn(categoryTypeList);

        // 3.数据拼装
        List<ProductVO> productVOList = new ArrayList<>();
        // 首先遍历类目
        for(ProductCategory productCategory : productCategoryList) {
            ProductVO productVO = new ProductVO();
            productVO.setCategoryType(productCategory.getCategoryType());
            productVO.setCategoryName(productCategory.getCategoryName());

            List<ProductInfoVO> productInfoVOList = new ArrayList<>();
            for (ProductInfo productInfo : productInfoList) {
                if (productInfo.getCategoryType().equals(productCategory.getCategoryType())) {
                    ProductInfoVO productInfoVO = new ProductInfoVO();
                    // setter太过麻烦，所以用Spring自带的BeanUtils，把productInfo的属性复制到productInfoVO中
                    BeanUtils.copyProperties(productInfo, productInfoVO);
                    productInfoVOList.add(productInfoVO);
                }
            }
            productVO.setProductInfoVOList(productInfoVOList);
            productVOList.add(productVO);
        }

        // ResultVO resultVO = new ResultVO();
        // ProductVO productVO = new ProductVO();
        // ProductInfoVO productInfoVO = new ProductInfoVO();
        // productVO.setProductInfoVOList(Arrays.asList(productInfoVO));
        // resultVO.setData(Arrays.asList(productVO));
        // resultVO.setData(productVOList);
        // resultVO.setCode(0);
        // resultVO.setMsg("成功");
        // return resultVO;
        return ResultVOUtils.success(productVOList);
    }
}
```

至此得到的json数据如下：<br>
![json数据](./screenshot/json4.jpg)

（中间过程类似以上，省略）

#### 订单详情OrderDetail和主订单OrderMaster相关操作（主要看Service层）
```
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {

    // 要查商品信息
    @Autowired
    private ProductService productService;

    @Autowired
    private OrderDetailRepository orderDetailRepository;

    @Autowired
    private OrderMasterRepository orderMasterRepository;

    @Override
    @Transactional /* 入库失败抛异常时回滚 */
    public OrderDTO create(OrderDTO orderDTO) {

        // OrderDTO是Service层和Controller层中间传递的对象，给Controller层调用的
        // Controller层不可能把所有数据传过来，传过来的话就不用Service层了，数据不能由前端传过来，要从数据库中取出来，尤其是商品单价

        // 订单创建时就生成订单Id
        String orderId = KeyUtil.genUniqueKey();
        BigDecimal orderAmount = new BigDecimal(0);

        // 传统写法
        // List<CartDTO> cartDTOList = new ArrayList<>();

        // 1.查询商品（数量，价格）：首先拿到商品列表，遍历
        for (OrderDetail orderDetail : orderDTO.getOrderDetailList()) {
            ProductInfo productInfo =  productService.findOne(orderDetail.getProductId());
            if (productInfo == null) {
                throw new SellException(ResultEnum.PRODUCT_NOT_EXIST);
            }

            // 2.计算订单总价（BigDecimal不能直接用* 的符号）
            orderAmount = productInfo.getProductPrice().multiply(new BigDecimal(orderDetail.getProductQuantity())).add(orderAmount);

            // 3.1 订单详情保存到数据库（OrderDetail）
            // 前端只传过来商品id和商品数量
            orderDetail.setDetailId(KeyUtil.genUniqueKey());
            orderDetail.setOrderId(orderId);
            // 商品图片和其他属性在ProductInfo里面
            // 把productInfo的属性复制到orderDetail里
            BeanUtils.copyProperties(productInfo, orderDetail);
            orderDetailRepository.save(orderDetail);

            // CartDTO cartDTO = new CartDTO(orderDetail.getProductId(), orderDetail.getProductQuantity());
            // cartDTOList.add(cartDTO);
        }

        // 3.2 订单主表入库
        OrderMaster orderMaster = new OrderMaster();
        // 这里要特别注意：如果orderDTO的属性值为null时，也会被拷贝，所以要先拷贝后设置，这样orderId才不会是null
        BeanUtils.copyProperties(orderDTO, orderMaster);
        orderMaster.setOrderId(orderId);
        orderMaster.setOrderAmount(orderAmount);
        // 设断点发现，orderStatus和payStatus在执行BeanUtils.copyProperties后被覆盖，所以要重新写回去
        orderMaster.setOrderStatus(OrderStatusEnum.NEW.getCode());
        orderMaster.setPayStatus(PayStatusEnum.WAIT.getCode());
        orderMasterRepository.save(orderMaster);

        // 4.下单成功的话扣库存（Lambda表达式），一旦入库失败就会抛异常
        List<CartDTO> cartDTOList = orderDTO.getOrderDetailList().stream().map(e ->
                new CartDTO(e.getProductId(), e.getProductQuantity()))
                .collect(Collectors.toList());
        productService.decreaseStock(cartDTOList);

        return orderDTO;
    }

    @Override
    public OrderDTO findOne(String orderId) {
        OrderMaster orderMaster = orderMasterRepository.findOne(orderId);
        if (orderMaster == null) {
            throw new SellException(ResultEnum.ORDER_NOT_EXIST);
        }
        List<OrderDetail> orderDetailList = orderDetailRepository.findByOrderId(orderId);
        if (CollectionUtils.isEmpty(orderDetailList)) {
            throw new SellException(ResultEnum.ORDERDETAIL_NOT_EXIST);
        }

        OrderDTO orderDTO = new OrderDTO();
        BeanUtils.copyProperties(orderMaster, orderDTO);
        orderDTO.setOrderDetailList(orderDetailList);

        return orderDTO;
    }

    @Override
    public Page<OrderDTO> findList(String buyerOpenid, Pageable pageable) {
        Page<OrderMaster> orderMasterPage = orderMasterRepository.findByBuyerOpenid(buyerOpenid, pageable);
        OrderDTO orderDTO = new OrderDTO();
        OrderMaster orderMaster = new OrderMaster();
        BeanUtils.copyProperties(orderMaster, orderDTO);
        List<OrderDTO> orderDTOList = OrderMaster2OrderDTOConverter.convert(orderMasterPage.getContent());
        // 把查到的OrderMaster转成OrderDTO
        Page<OrderDTO> orderDTOPage = new PageImpl<OrderDTO>(orderDTOList, pageable, orderMasterPage.getTotalElements());
        return orderDTOPage;
    }

    @Override
    @Transactional
    public OrderDTO cancel(OrderDTO orderDTO) {
        OrderMaster orderMaster = new OrderMaster();
        BeanUtils.copyProperties(orderDTO, orderMaster);
        // 先判断订单状态
        if (!orderDTO.getOrderStatus().equals(OrderStatusEnum.NEW.getCode())) {
            log.error("【取消订单】订单状态不正确，orderId={}, orderStatus={}", orderDTO.getOrderId(), orderDTO.getOrderStatus());
            throw new SellException(ResultEnum.ORDER_STATUS_ERROR);
        }

        // 修改订单状态
        orderDTO.setOrderStatus(OrderStatusEnum.CANCEL.getCode());
        BeanUtils.copyProperties(orderDTO, orderMaster);
        OrderMaster updateResult = orderMasterRepository.save(orderMaster);
        if (updateResult == null) {
            log.error("【取消订单】更新失败，orderMaster={}", orderMaster);
            throw new SellException(ResultEnum.ORDER_UPDATE_FAIL);
        }

        // 返回库存
        if (CollectionUtils.isEmpty(orderDTO.getOrderDetailList())) {
            log.error("【取消订单】订单中无商品详情，orderDTO={}", orderDTO);
            throw new SellException(ResultEnum.ORDER_DETAIL_EMPTY);
        }
        // 直接构造一个list
        List<CartDTO> cartDTOList = orderDTO.getOrderDetailList().stream().map( e ->
            new CartDTO(e.getProductId(), e.getProductQuantity())
        ).collect(Collectors.toList());
        productService.increaseStock(cartDTOList);

        // 如果已支付，需要退款
        if (orderDTO.getPayStatus().equals(PayStatusEnum.SUCCESS.getCode())) {
            // TODO
        }
        return orderDTO;
    }

    @Override
    @Transactional
    public OrderDTO finish(OrderDTO orderDTO) {
        // 第一步同样要判断订单
        if (!orderDTO.getOrderStatus().equals(OrderStatusEnum.NEW.getCode())) {
            log.error("【完结订单】订单状态不正确, orderId={], orderStatus={}", orderDTO.getOrderId(), orderDTO.getOrderStatus());
            throw new SellException(ResultEnum.ORDER_STATUS_ERROR);
        }

        // 然后修改订单状态
        orderDTO.setOrderStatus(OrderStatusEnum.FINISHED.getCode());
        OrderMaster orderMaster = new OrderMaster();
        BeanUtils.copyProperties(orderDTO, orderMaster);
        OrderMaster updateResult = orderMasterRepository.save(orderMaster);
        if (updateResult == null) {
            log.error("【完结订单】更新失败, orderMaster={}", orderMaster);
            throw new SellException(ResultEnum.ORDER_UPDATE_FAIL);
        }
        return orderDTO;
    }

    @Override
    @Transactional
    public OrderDTO paid(OrderDTO orderDTO) {
        // 必须是新订单，并且是等待支付才能执行支付
        // 判断订单状态
        if (!orderDTO.getOrderStatus().equals(OrderStatusEnum.NEW.getCode())) {
            log.error("【订单支付】订单状态不正确, orderId={], orderStatus={}", orderDTO.getOrderId(), orderDTO.getOrderStatus());
            throw new SellException(ResultEnum.ORDER_STATUS_ERROR);
        }

        // 修改支付状态
        if (!orderDTO.getPayStatus().equals(PayStatusEnum.WAIT.getCode())) {
            log.error("【订单支付】订单支付状态不正确, orderDTO={}", orderDTO);
            throw new SellException(ResultEnum.ORDER_PAY_STATUS_ERROR);
        }

        // 修改订单状态
        orderDTO.setPayStatus(PayStatusEnum.SUCCESS.getCode());
        OrderMaster orderMaster = new OrderMaster();
        BeanUtils.copyProperties(orderDTO, orderMaster);
        OrderMaster updateResult = orderMasterRepository.save(orderMaster);
        if (updateResult == null) {
            log.error("【订单支付】更新失败, orderMaster={}", orderMaster);
            throw new SellException(ResultEnum.ORDER_UPDATE_FAIL);
        }
        return orderDTO;
    }
}
```