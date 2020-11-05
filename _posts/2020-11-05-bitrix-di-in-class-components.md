---
title: БУС и инъекция зависимостей в классовые компоненты 
---

В 2020 году будет тратой времени объяснять, что такое dependency injection и какую пользу построение ядра вокруг
контейнера зависимостей приносит современному PHP-проекту. Программистам, полноценно работающим
с платформой 1С-Битрикc, дополнительно нет смысла рассказывать, какой процент стандартных модулей 
платформы этот паттерн применяет (на всякий случай - ноль, нет его там в принципе).

Но, несмотря на то, что нет - хочется. А если хочется, то стоит попробовать,
независимо от того, _что_ получится в итоге.

## Введение

В целом, весь код проекта на БУС можно разделить на 3 группы:

1. Модули ядра. Не будет использовать DI никогда.
   
2. Изолированные модули и скрипты интегратора - до тех пор, пока мы не взаимодействуем с API ядра,
   построить взаимодействие на основе контейнера, как стороннего решения из числа популярных,
   так и самописного не составляет труда.
   
3. Классы интегратора на основе классов ядра, загружаемые и используемые ядром по особой логике.
   Самый распространенный пример - классовые компоненты, наследуемые от `\CBitrixComponent`.
   
С п. 1 ничего не сделать, как по причине проблем с обновлениями, так и объемом работ, даже в случае ограниченных правок.

С п.2 делать ничего не нужно.

П. 3 представляет наибольший интерес с практической точки зрения. Де-факто компонент - это фронт контроллер
с функционалом роутера (случай комплексного компонента). В случае с реализацией интерфейса `Controllerable`, также 
со способностью декларативного описания поддерживаемых действия и подключения middleware - то есть, почти индустриальный
стандарт. Осталось добавить возможность инъекции инстансов, для начала в конструктор, и программистам на Laravel будет
нечего противопоставить мощи двухсот мегабайт отечественного кода:

```php
class FooComponent extends \CBitrixComponent
{
     /**
         * @param \CBitrixComponent|null $component
         * @throws \LoaderException
         */
        public function __construct(
            ?\CBitrixComponent $component = null,
            ?\SomeService $service = null
        ) {
            parent::__construct($component);
    
            $this->service = $service;            
        }
}
```
Осталось понять, как это реализовать.

## Анализ

На первый взгляд - никак. Но, чтобы быть уверенными, пройдем с отладчиком, чтобы заявлять об этом
с уверенностью и полным знанием процесса загрузки компонентов.

* Точка входа - вызов метод `IncludeComponent()` синглтона `$APPLICATION`:

    ```php
    $APPLICATION->IncludeComponent(
        "custom:system.empty", 
        "index_welcome", 
        [],
        false
    );
    ```
  
* Метод проверяет параметры, флаги отладки, вычисляет путь к компоненту по его имени, создает
  пустой инстанс `\CBitrixComponent`, инициализирует его и выполняет собственно подключение компонента через
  его нестатический метод `IncludeComponent()` :
  
  ```php
  public function IncludeComponent(
      $componentName,
      $componentTemplate,
      $arParams = array(),
      $parentComponent = null,
      $arFunctionParams = array(),
      $returnResult = false
  ) {
      // ...
      $component = new CBitrixComponent();
      if ($component->InitComponent($componentName)) {
          // ...
          if ($bComponentEnabled) {
              // ...
              $result = $component->IncludeComponent(
                  $componentTemplate,
                  $arParams,
                  $parentComponent,
                  $returnResult
              );
              // ...
          }
          // ...
      }
      // ...
  }
  ```
  
* Ключевые моменты инициализации и подключения выглядят так:

  ```php
    final public function initComponent($componentName, $componentTemplate = false)
    {
        $path2Comp = CComponentEngine::MakeComponentPath($componentName);
            $componentPath = getLocalPath("components".$path2Comp);
            $this->classOfComponent = self::__getClassForPath($componentPath);
    }

    final public function includeComponent(
        $componentTemplate,
        $arParams,
        $parentComponent,
        $returnResult = false
    ) {
        if ($parentComponent instanceof CBitrixComponent) {
            $this->__parent = $parentComponent;
        }
  
        if ($this->classOfComponent) {
            $component = new $this->classOfComponent($this);
  
            $component->arParams = $component->onPrepareComponentParams($arParams);
            $component->__prepareComponentParams($component->arParams);
  
            if ($returnResult) {
                $component->executeComponent();
                $result = $component->arResult;
            } else {
                $result = $component->executeComponent();
           }
        } else {
            // Работа с бесклассовыми компонентами 
        }
  
        return $result;
    }
    ```
  
  Прямой вызов оператора `new` с единственным параметром. Совершенно ясно, что в этом методе ловить нечего.
  Но класс компонента создается динамичеcки, и на основе не константы, а классового свойства, вычисляемого же
  статическим (!) методом `__getClassForPath`. Взглянем туда:
    
  ```php
  final private function __getClassForPath($componentPath)
  {
     if (!isset(self::$__classes_map[$componentPath])) {
         $fname = $_SERVER["DOCUMENT_ROOT"] . $componentPath . "/class.php";
 
         if (file_exists($fname) && is_file($fname)) {
             $beforeClasses = get_declared_classes();
             $beforeClassesCount = count($beforeClasses);
             include_once($fname);
             $afterClasses = get_declared_classes();
             $afterClassesCount = count($afterClasses);
 
             for ($i = $beforeClassesCount; $i < $afterClassesCount; $i++) {
                 if (!isset(self::$classes[$afterClasses[$i]])
                     && is_subclass_of($afterClasses[$i], "cbitrixcomponent")
                 ) {
                     if (!isset(self::$__classes_map[$componentPath])
                         || is_subclass_of(
                                $afterClasses[$i],
                                self::$__classes_map[$componentPath]
                            )
                     ) {
                         self::$__classes_map[$componentPath] = $afterClasses[$i];
                         self::$classes[$afterClasses[$i]] = true;
                     }
                 }
             }
         } else {
             self::$__classes_map[$componentPath] = "";
         }
     }
 
     return self::$__classes_map[$componentPath];
   }
   ```
  
  Метод вычисляет, какой же именно класс из подключившихся при загрузке файла, нужно
  интерпретировать как компонент. Итоговый класс заносится в кеш `$__classes_map` -
  это статическое свойство, значение которого в ходе нормальной работы ядра никогда не
  определяется.

Итог анализа: мы совершенно точно не можем сделать так, чтобы \CBitrixComponent внедрял зависимости
в конструктор конкретного класса компонента при его инстанциации без модификации ядра .

Но еще раз перечитаем ТЗ и посмотрим на код свежим взглядом - вопрос-то не в этом.
Не в строгой реализации некоторого паттерна, а в том, чтобы эти зависимости компонент так или иначе получал.

Это возможно в случае, если мы контролируем вызов `new`,
и, помимо инстанциации экземпляра самого класса, есть ровно два варианта,
как можно этого добиться - наследование или делегирование. Подмена класса компонента классом
наследника, который сможет разрешать значения параметров конструктора родителя,
либо же классом, создающем хранящий экземпляр компонента на своих условиях.
хранящем его в своем `private`-свойстве и делегирующем в него вызовы методов компонента.

Вариант 1 проще в реализации и жизнеспособнее за счет поддержки полиморфизма - остановимся на нём. 

## Реализация
      
### Шаг 1: Подмена имени класса

Карта классов, как уже упоминалось, хранится в `private static self::$__classes_map`
с начальным значением в виде пустого массива. Заменим значение этого свойства на наследник `\ArrayObject`,
реализующий нужную нам логику `offsetGet` или `offsetSet`. Саму же замену произведем с помощью особенностей
класса \Closure:

```php
function invoke_internal($instance, \Closure $closure)
{
    return $closure
        ->bindTo($instance, get_class($instance))
        ->call($instance);
}

invoke_internal(new \CBitrixComponent(), function () {
    $agent = new ComponentInterceptorAgent(static::$__classes_map);
    static::$__classes_map = $agent;
});
``` 

Код заменяющего класса в сокращенном виде: 

```php
class ComponentInterceptorAgent extends \ArrayObject
{
    /**
     * @param array $input
     * @param int $flags
     * @param string $iterator_class
     */
    public function __construct($input = array(), $flags = 0, $iterator_class = "ArrayIterator")
    {
        parent::__construct($input, $flags, $iterator_class);

        foreach ($input as $componentPath => $componentClass) {
            $this->classCache[$componentClass] = $this[$componentPath];
        }
    }

    /**
     * @param string $componentPath
     * @return mixed
     * @throws \ReflectionException
     */
    public function offsetGet($componentPath)
    {
        $componentClass = parent::offsetGet($componentPath);

        if (!$componentClass) {
            $this[$componentPath] = $componentClass;
            return $componentClass;
        }

        if (in_array($componentClass, $this->wrapperCache)) {
            return $componentClass;
        }

        $predicate = $this->options->getPredicate();

        if (isset($predicate) && !$predicate($componentClass)) {
            $this[$componentPath] = $componentClass;
            return $componentClass;
        }

        $parentClass = Classname::from($componentClass);
        $wrapperClass = $this->getWrapperClass($parentClass);

        if (!isset($this->classCache[$componentClass])) {
            $reflection = new \ReflectionClass($componentClass);

            if ($reflection->isFinal()) {
                $this->classCache[$componentClass] = $componentClass;
                $this[$componentPath] = $componentClass;
            } else {
                $options = new WrapperOptions(
                    $wrapperClass,
                    $parentClass,
                    $this->options->getMethodDelegate()
                );

                $builder = new WrapperBuilder();
                $wrapper = $builder->buildWrapperComponent($options);
                $evaluator = new WrapperDeployer();
                $evaluator->deployWrapperClass($wrapper);

                $this->classCache[$componentClass] = (string) $wrapperClass;
                $this->wrapperCache[] = (string) $wrapperClass;
                $this[$componentPath] = (string) $wrapperClass;
            }
        }

        return $this->classCache[$componentClass];
    }
}
```

Как видно из кода, переопределяется метод чтения из массива - можно было переопределить `offsetSet`,
это вопрос предпочтений.

Метод проверяет, не является ли класс подключаемого компонента `final`. Если нет - создает наследника,
заносит его имя в кеш (чтобы не уйти в рекурсию), и возвращает его имя.

Важный момент - компонент собирается динамически, но мы не можем заранее предусмотреть,
каким именно методы нам покажется необходимым переопределить в наследнике. Поэтому в метод сборки компонента
передается внешний _делегат_ - см. вызов `$this->options->getMethodDelegate()`. Делегат определяет методы,
особым образом аннотирует их, и при вызове метода виртуального наследника вызов передается к делегату.
Сделано это исключительно ради удобства и универсальности.

### Шаг 2: Создание прокси-наследника

Классы наследников будут создаваться динамически - это реализуемо благодаря функции `eval()`.
Любой код, выполняемый интерпретатором из файла, может также быть выполнен в `eval()`, причем переданный код
будет интерпретирован как содержимое отдельного PHP-файла, как если бы он подключился через `include` - это
позволяет создавать runtime-классы в нужном нам пространстве имен
(в данном случае аналогичном родителю, во избежание конфликтов).

Основной момент (цитата из строки, которая в итоге будет передана в `eval`):

```php
public function __construct(...\$args)
{
    if (method_exists(get_parent_class(\$this), '__construct')) {
        \$args = {$this->getPreConstructInvocation($options)} ?: \$args;
    
        parent::__construct(...\$args);
    }
    
    {$this->getPostConstructInvocation($options)}
}
``` 

Мы не можем куда-либо делегировать вызов конструктора, поэтому делегируем обработку аргументов.
Результирующий масссив со всеми сервисами будет передан в родительский конструктор.  

Остальной код сборщика классов цитировать нет смысла, т.к. он достаточно прост.
Полный текст класса [доступен](https://github.com/whiskyjs/bx-mutagen/blob/develop/src/Core/Delegation/WrapperBuilder.php) на GitHub.

### Шаг 3: Создание контейнера зависимостей

Существует несколько готовых реализаций, но для иллюстрации концепта оказалось проще и быстрее написать свою,
намного более примитивную и ошибочную. Т.к. инстанс на момент перехвата уже существует,
container()->make() и аналоги неприменимы, и необходим публичный метод разрешение значений аргументов класса:

```php
   public function resolveMethodArguments(
       string $class,
       $method,
       array $arguments
   ): array {
      return $this->__resolveMethodArguments($class, $method, $arguments);
   }

    private function __resolveMethodArguments(
        string $class,
        $method,
        array $arguments,
        array $chain = []
    ): array {
        $args = [];

        if (is_string($method)) {
            $method = new \ReflectionMethod($class, $method);
        }

        foreach ($method->getParameters() as $parameter) {
            $parameterType = $parameter->getType();
            $parameterTypeName = $parameterType ? $parameterType->getName() : null;
            $parameterValue = null;

            if (is_map($arguments) && array_key_exists($parameter->getName(), $arguments)) {
                $parameterValue = $arguments[$parameter->getName()];
                unset($arguments[$parameter->getName()]);
            } elseif (($arguments && !$parameterTypeName)
                || ($parameterTypeName === get_class($arguments[0] ?? null))
            ) {
                if ($parameter->isVariadic()) {
                    $parameterValue = array_splice($arguments, 0);
                } else {
                    $parameterValue = array_shift($arguments);
                }
            } elseif ($parameterTypeName) {
                if ($class === $parameterTypeName) {
                    // Класс не может требовать экземпляр самого себя в конструкторе
                    throw new ResolutionError(
                        "Невозможно разрешить зависимость: циклическая зависимость."
                    );
                }

                $parameterValue = $this->__make(
                    $parameterTypeName,
                    [],
                    array_merge($chain, [$class])
                );
            } elseif ($parameter->isDefaultValueAvailable()) {
                $parameterValue = $parameter->getDefaultValue();
            }

            if ($parameter->isVariadic()) {
                $args = array_merge($args, $parameterValue);
            } else {
                $args[] = $parameterValue;
            }
        }

        return $args;
    }
```

### Шаг 4: Тесты

1. Класс сервиса:

    ```php
    class SomeService
    {
        public function foo(): string
        {
            return "bar";
        }
    }
    ```
   
2. Binding в контейнер:

   ```php
   container()->singleton(SomeService::class, function () {
       return new SomeService();
   }); 
   ```

3. Подключение перехватчика компонентов:

   ```php
    class DIDelegate extends Delegate
    {
        /**
         * @bx-delegate
         * @param DelegationOrigin $origin
         * @param $arParams
         * @return mixed
         */
        public function preConstruct($origin, ...$args): array
        {
           // apply() === invoke_internal() без необходимости указывать объект 
            return $origin->apply(function () use ($args) {
                return container()->resolveMethodArguments(
                   get_parent_class($this), "__construct", $args
                );
            });
        }
    
        /**
         * @bx-delegate
         * @param $arParams
         * @return void
         */
        public function postConstruct($origin, ...$args)
        {
        }
    }
    
    $myDelegate = new DIDelegate();
    
    $options = new ComponentInterceptorOptions($myDelegate);
    $interceptor = new ComponentInterceptorPlugin($options);
    $interceptor->plugIn();
   ```
   
4. Компонент:
   
   ```php
   class SubscribeFormComponent extends RestComponent
   {
       public function __construct(
           ?\CBitrixComponent $component = null,
           ?\SomeService $service = null
       ) {
           parent::__construct($component);
   
           $value = $service->foo(); // "bar"
       }
   }
   ```

## Итоги

* Программисты на Laravel грустны и подавлены, и это мы еще только начали.
  В следующей статье - сборка своего Artisan, с динамическим набором команд и скрытием пространств имен.
  
* Даже если все говорят, что что-то невозможно в принципе, иногда оно всё-таки возможно - и обычно попытка стоит того,
  даже если на выходе получается всего лишь забавная статья в блог.


## Ссылки

Composer-пакет bx-mutagen, реализующий функционал перехвата компонентов:

[https://github.com/whiskyjs/bx-mutagen](https://github.com/whiskyjs/bx-mutagen)
