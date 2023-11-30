## 通过Attribute实现Symfony请求自动验证
Symfony框架用了一段时间后感觉验证组件还是太难用了，相比Laravel里面可以在controller里面直接调用```$this->validate($request, [])```手动验证数据，Symfony里面只能实现把验证规则绑定在Entity的字段上，然后在form里面进行验证，虽然也可以手动调用validator实例进行验证，但是代码写起来十分麻烦，我想用Attribute把验证规则挂载到controller里面的方法上，把规则写到Attribute里面，实现自动验证，试试

首先，可以构建一个Attribute类，这个Attribute类用来定义需要对请求验证的字段，收集验证规则
```php
#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
class ValidateRequestParam
{
    public function __construct(private string $field, private Constraint|array $constraint = [])
    {
    }

    public function getField(): string
    {
        return $this->field;
    }

    public function getConstraints(): array
    {
        $constraints = [];
        if ($this->constraint instanceof Constraint) {
            $constraints[] = $this->constraint;
        } else {
            foreach ($this->constraint as $constraint) {
                if ($constraint instanceof Constraint) {
                    $constraints[] = $constraint;
                }
            }
        }

        return $constraints;
    }
}
```
因为是挂载在controller的方法上的，而且可能会应用到多个字段，所以必须选择```Attribute::TARGET_METHOD```以及```Attribute::IS_REPEATABLE```

接下来就是监听```kernel.request```事件，定义一个EventSubscriber，我们可以在$event中通过反射获取当前方法，并获得挂载的Attribute实例，然后调用框架绑定的validator服务，手动的验证request中对应的数据。
但是这样搞有个问题，就是创建反射是十分消耗性能的，幸运的是symfony有编译缓存的，那么我们可以在编译期扫描所有的方法，然后创建一个```$methodValidationsMap```，它是一个controller::method => $rules的缓存，接下来在事件监听中从这个map中获取对应的规则，这样在```kener.request```中就不用再创建反射了，可以提升性能。

那么我们需要创建一个CompilerPass，这个CompilerPass会创建并缓存我们需要的```$methodValidationsMap```
```php
class ValidateRequestParamPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container)
    {
        if (!$container->hasDefinition(ValidateRequestSubscriber::class)) {
            throw new LogicException('validation subscriber not configured');
        }
        $taggedServices = $container->findTaggedServiceIds('controller.service_arguments');
        $serviceDefinitions = array_map(fn (string $id) => $container->getDefinition($id), array_keys($taggedServices));
        $methodValidationsMap = [];
        foreach ($serviceDefinitions as $serviceDefinition) {
            $controllerClass = $serviceDefinition->getClass();
            $refClass = $container->getReflectionClass($controllerClass);
            foreach ($refClass->getMethods(ReflectionMethod::IS_PUBLIC | ~ReflectionMethod::IS_STATIC) as $refMethod) {
                $attributes = $refMethod->getAttributes(ValidateRequestParam::class);
                if (!count($attributes)) {
                    continue;
                }
                $fieldRuleMaps = [];
                foreach ($attributes as $attribute) {
                    /** @var ValidateRequestParam $attributesInstance */
                    $attributesInstance = $attribute->newInstance();
                    $field = $attributesInstance->getField();
                    $fieldRuleMaps[$field]= $attributesInstance->getConstraints();
                }
                // 通过controller和method创建方法的联合key
                $classMapKey = sprintf('%s::%s', $serviceDefinition->getClass(), $refMethod->getName());
                // 注意编译缓存只能缓存字符串及数组，无法缓存对象，所以这里必须将数据序列化后缓存
                $methodValidationsMap[$classMapKey] = serialize($fieldRuleMaps);
            }
        }
        // ValidateRequestSubscriber的构造函数将接收我们map作为参数
        $container->getDefinition(ValidateRequestSubscriber::class)->setArgument('$methodValidationsMap', $methodValidationsMap);
    }
}
```

```$methodValidationsMap```保存了所有方法到对应的字段及验证规则的映射，那么我们可以开始创建```ValidateRequestSubscriber```了。```ValidateRequestSubscriber```如下。

```php
class ValidateRequestSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private array $methodValidationsMap,
        private ValidatorInterface $validator,
    ) {
    }

    public function onKernelRequest(RequestEvent $event): void
    {
        $request = $event->getRequest();
        $controllerMethod = $request->attributes->get('_controller');
        if (isset($this->methodValidationsMap[$controllerMethod])) {
            $serializedValidationGroups = $this->methodValidationsMap[$controllerMethod];
            /** @var array<string, list<Constraint>> $validationGroups */
            $validationGroups = unserialize($serializedValidationGroups, ['allowed_classes'=>true, 'max_depth'=>5]);
            if (!empty($validationGroups)) {
                foreach ($validationGroups as $field => $rules) {
                    foreach ($rules as $rule) {
                        if (!$rule instanceof Constraint) {
                            continue;
                        }
                        $errors = $this->validator->validate($request->get($field), $rule);
                        if (count($errors)) {
                            //todo 抛出验证异常
                        }
                    }
                }
            }
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'kernel.request' => 'onKernelRequest',
        ];
    }
}
```
在controller里面的用法
```php
class UserController
{
    #[ValidateRequestParam('firstname', [new Assert\Length(max: 255), new Assert\NotBlank(), true])]
    #[ValidateRequestParam('middlename', [new Assert\Length(max: 255)], false)]
    #[ValidateRequestParam('lastname', [new Assert\Length(max: 255), new Assert\NotBlank()], true)]
    #[ValidateRequestParam('twitter', [new Assert\Length(max: 255)], false)]
    #[ValidateRequestParam('facebook', [new Assert\Length(max: 255)], false)]
    #[ValidateRequestParam('linkedin', [new Assert\Length(max: 255)], false)]
    #[ValidateRequestParam('organization', [new Assert\Length(max: 255)], false)]
    #[ValidateRequestParam('research_interest', [new Assert\Length(max: 255), new Assert\NotBlank()], true)]
    public function editProfile(Request $request){
        //todo 数据已经验证过，放心使用
    }
}
```

大概就是这样了，代码看着舒服了很多也简单了很多。

这段代码也只是简单的示例版本，还有些漏洞，比如没有考虑到数据```$request```中对应字段数据不存在的问题，而且它只能对简单类型进行验证，像entity.Id这些验证功能也需要兼容，并且最好支持id到entity的自动转换。这个还有待探索