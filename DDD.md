

#### 一、接口层
1、主要功能：
- 网络协议的转化：通常这个已经由各种框架给封装掉了，我们需要构建的类要么是被注解的bean，要么是继承了某个接口的bean。
- 统一鉴权：比如在一些需要AppKey+Secret的场景，需要针对某个租户做鉴权的，包括一些加密串的校验
- Session管理：一般在面向用户的接口或者有登陆态的，通过Session或者RPC上下文可以拿到当前调用的用户，以便传递给下游服务。
- 限流配置：对接口做限流避免大流量打到下游服务
- 前置缓存：针对变更不是很频繁的只读场景，可以前置结果缓存到接口层
- 异常处理：通常在接口层要避免将异常直接暴露给调用端，所以需要在接口层做统一的异常捕获，转化为调用端可以理解的数据格式
- 日志：在接口层打调用日志，用来做统计和debug等。一般微服务框架可能都直接包含了这些功能。


#### 2、返回值和异常处理规范，Result vs Exception
 > Interface层的HTTP和RPC接口，返回值为Result，捕捉所有异常
 > Application层的所有接口返回值为DTO，不负责处理异常

网上例子：通过切面+注解的方式实现异常捕获，但也可以使用全局异常的方式。

- 例子一：切面+注解
``` java 
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ResultHandler {

}

@Aspect
@Component
public class ResultAspect {
    @Around("@annotation(ResultHandler)")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        Object proceed = null;
        try {
            proceed = joinPoint.proceed();
        } catch (ConstraintViolationException cve) {
            return Result.fail(cve.getMessage());
        } catch (Exception e) {
            return Result.fail(e.getMessage());
        }
        return proceed;
    }
}

```

  - interface层代码
``` java

@PostMapping("checkout")
@ResultHandler
public Result<OrderDTO> checkout(Long itemId, Integer quantity) {
    CheckoutCommand cmd = new CheckoutCommand();
    OrderDTO orderDTO = checkoutService.checkout(cmd);
    return Result.success(orderDTO);
}
```

- 例子二，全局异常，使用注解


#### 3、接口层的接口的数量和业务间的隔离
当支撑的上游业务比较多时，刻意去追求接口的统一通常会导致方法中的参数膨胀，或者导致方法的膨胀。举个例子：假设有一个宠物卡和一个亲子卡的业务公用一个开卡服务，但是宠物需要传入宠物类型，亲子的需要传入宝宝年龄。

``` java
// 可以是RPC Provider 或者 Controller
public interface CardService {

    // 1）统一接口，参数膨胀
    Result openCard(int petType, int babyAge);

    // 2）统一泛化接口，参数语意丢失
    Result openCardV2(Map<String, Object> params);

    // 3）不泛化，同一个类里的接口膨胀
    Result openPetCard(int petType);
    Result openBabyCard(int babyAge);
}
```

- 建议：
> 一个Interface层的类应该是“小而美”的，应该是面向“一个单一的业务”或“一类同样需求的业务”，需要尽量避免用同一个类承接不同类型业务的需求。

基于上面的这个规范，可以发现宠物卡和亲子卡虽然看起来像是类似的需求，但并非是“同样需求”的，可以预见到在未来的某个时刻，这两个业务的需求和需要提供的接口会越走越远，所以需要将这两个接口类拆分开。再举个例子，上报虚拟机和上报物理机接口，这两个接口的字段、业务场景不一样，也是需要将两个接口类进行拆分。
``` java

public interface PetCardService {
    Result openPetCard(int petType);
}

public interface BabyCardService {
    Result openBabyCard(int babyAge);
}
```

类的职责明确，符合了Single Responsibility Principle单一职责原则，也就是说一个接口类仅仅会因为一个（或一类）业务的变化而变化。一个建议是当一个现有的接口类过度膨胀时，可以考虑对接口类做拆分，拆分原则和SRP一致。
也许会有人问，如果按照这种做法，会不会产生大量的接口类，导致代码逻辑重复？答案是不会，因为在DDD分层架构里，接口类的核心作用仅仅是协议层，每类业务的协议可以是不同的，而真实的业务逻辑会沉淀到应用层。也就是说Interface和Application的关系是多对多的：
![image](https://user-images.githubusercontent.com/32328586/119536758-0298c600-bdbc-11eb-8d35-89d5d65f0a08.png)

### 二、Application层

#### 1、Application层的组成部分
- ApplicationService应用服务：最核心的类，负责业务流程的编排，核心功能是承接“业务流程“，但本身不负责任何业务逻辑。
- DTO Assembler：负责将内部领域模型转化为可对外的DTO。
- Command、Query、Event对象：作为ApplicationService的入参。
- 返回的DTO：作为ApplicationService的出参。

#### 2、Command、Query、Event对象
- Command指令：指调用方明确想让系统操作的指令，其预期是对一个系统有影响，也就是写操作。通常来讲指令需要有一个明确的返回值（如同步的操作结果，或异步的指令已经被接受）。
- Query查询：指调用方明确想查询的东西，包括查询参数、过滤、分页等条件，其预期是对一个系统的数据完全不影响的，也就是只读操作。
- Event事件：指一件已经发生过的既有事实，需要系统根据这个事实作出改变或者响应的，通常事件处理都会有一定的写操作。事件处理器不会有返回值。这里需要注意一下的是，Application层的Event概念和Domain层的DomainEvent是类似的概念，但不一定是同一回事，这里的Event更多是外部一种通知机制而已。

![image](https://user-images.githubusercontent.com/32328586/119539764-2f021180-bdbf-11eb-8433-cc3ff3495d42.png)

#### 3、ApplicationService
只做编排，不做业务逻辑处理，相关的业务参数校验、计算逻辑，可以放到Entity或者Domain Service里面如：
``` java

@Data
public class Order {

    private Long itemUnitPrice;
    private Integer count;

    // 把原来一个在ApplicationService的计算迁移到Entity里
    public Long getTotalCost() {
        return itemUnitPrice * count;
    }
}

order.setItemUnitPrice(item.getPriceInCents());
order.setCount(cmd.getQuantity());
```

常见的用法如下：
- 准备数据：包括从外部服务或持久化源取出相对应的Entity、VO以及外部服务返回的DTO。
- 执行操作：包括新对象的创建、赋值，以及调用领域对象的方法对其进行操作。需要注意的是这个时候通常都是纯内存操作，非持久化。
- 持久化：将操作结果持久化，或操作外部系统产生相应的影响，包括发消息等异步操作。

注意：
> ApplicationService应该永远返回DTO而不是Entity。

#### 4、防腐层
无防腐层的情况：
![image](https://user-images.githubusercontent.com/32328586/119546218-4395d800-bdc6-11eb-8172-749813cb38e0.png)

有防腐层的情况：
![image](https://user-images.githubusercontent.com/32328586/119546233-485a8c00-bdc6-11eb-94ce-e0c97d28afb5.png)

代码举例：
``` java
// 自定义的内部值类
@Data
public class ItemDTO {
    private Long itemId;
    private Long sellerId;
    private String title;
    private Long priceInCents;
}

// 商品Facade接口
public interface ItemFacade {
    ItemDTO getItem(Long itemId);
}
// 商品facade实现
@Service
public class ItemFacadeImpl implements ItemFacade {

    @Resource
    private ExternalItemService externalItemService;

    @Override
    public ItemDTO getItem(Long itemId) {
        ItemDO itemDO = externalItemService.getItem(itemId);
        if (itemDO != null) {
            ItemDTO dto = new ItemDTO();
            dto.setItemId(itemDO.getItemId());
            dto.setTitle(itemDO.getTitle());
            dto.setPriceInCents(itemDO.getPriceInCents());
            dto.setSellerId(itemDO.getSellerId());
            return dto;
        }
        return null;
    }
}

// 库存Facade
public interface InventoryFacade {
    boolean withhold(Long itemId, Integer quantity);
}
@Service
public class InventoryFacadeImpl implements InventoryFacade {

    @Resource
    private ExternalInventoryService externalInventoryService;

    @Override
    public boolean withhold(Long itemId, Integer quantity) {
        return externalInventoryService.withhold(itemId, quantity);
    }
}
```


ApplicationService变为：
``` java
@Service
public class CheckoutServiceImpl implements CheckoutService {

    @Resource
    private ItemFacade itemFacade;
    @Resource
    private InventoryFacade inventoryFacade;
    
    @Override
    public OrderDTO checkout(@Valid CheckoutCommand cmd) {
        ItemDTO item = itemFacade.getItem(cmd.getItemId());
        if (item == null) {
            throw new IllegalArgumentException("Item not found");
        }

        boolean withholdSuccess = inventoryFacade.withhold(cmd.getItemId(), cmd.getQuantity());
        if (!withholdSuccess) {
            throw new IllegalArgumentException("Inventory not enough");
        }

        // ...
    }
}
```

进一步的话，可以把判空在ApplicationService里面抽取独立方法。


