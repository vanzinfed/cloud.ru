# cloud.ru

## 1. Вопросы для разогрева</h1>
  1. В области безопасности я сталкивался с шифрованием данных. В своем резюме я вкратце описывал свой проект на Arduino, Lora, ESP, где задача заключалась в передаче GPS - данных. Эти данные было легко перехватить, подобрав   ту частоту на которой работает Lora, и изучить, так как данные не были зашифрованы.
  2. Security code review проводил в рамках университетских проектов, где требовалось закрывать различные “дыры” для полной работоспособности программы.
  3.	Сканировал различные сайты на наличие SQL- инъекций с помощью автоматизированной программы **Acunetix**. 
  Для начала осуществлялся поиск предпологаемо уязвимых страниц на сайте. После чего URL закидывался в сканер, где находились SQL – инъекции по параметру. После чего требовалось переходить в **sqlmap**  и по целевому запросу       проводить аудит.<br/>
  Также находились сайты с XSS уязвимостью. В поле поиска внедрялись теги "img" с ссылкой на картинку. Так,  можно было передать ссылку сайта другому пользователю с этой самой картинкой и она также будет видна ему. Если  внести вредоносный код в поле поиска, то он исполниться на компьютере другого пользователя при открытии ссылки.
  4. Для меня это отличная возможность связать свою жизнь в сфере информационной безопасности и стать частью вашей замечательной команды в качестве AppSec-инженера.<br/>
Работа в компании cloud.ru, лидере в области облачных вычислений представляется мне уникальной возможностью применить и расширить мои знания в реальных проектах, способствуя созданию надежных и безопасных решений для ваших клиентов. Я особенно заинтересован в разработке и укреплении безопасности гибридных и частных облаков, что, на мой взгляд, является ключом к защите будущих технологических инноваций. 

С нетерпением ожидаю возможности учиться у и сотрудничать с ведущими специалистами в области AppSec, развивая безопасные и инновационные решения для cloud.ru и ваших клиентов.

---

## 2. Security code review

### Часть 1. Security code review: GO

 - Уязвимость: SQL Injection.
 - Строки:
```
query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
rows, err := db.Query(query)
```
- Последствия: В данной строке возможна эксплуатация SQL уязвимости  которая может привести к компрометации базы данных. Злоумышленник может выполнить произвольные SQL-запросы, получить, изменить или удалить данные из   базы данных.
  Например: возьмем в качестве примера такой ввод пользователя:
  ```
  searchQuery = "'; DROP TABLE products; --"
  ```
  Сформированный запрос после форматирования:
  ```
  SELECT * FROM products WHERE name LIKE '%'; DROP TABLE products; --%'
  ```
  В результате выполнения этого запроса в базе данных будет удалена таблица `products`.

- Решение:
  ```
  query := "SELECT * FROM products WHERE name LIKE ?":
  rows, err := db.Query(query, "%"+searchQuery+"%")
  ```
  Вместо конкретного значения в запросе, используется знак вопроса (?), который является плейсхолдером для значений, что гарантируют, что ввод пользователя будет обработан исключительно как данные, а не как часть SQL-    кода. Это исключает возможность выполнения произвольных SQL-команд злоумышленником и является безопасным способом передачи значений в SQL запрос.

### Часть 2: Security code review: Python
  **Пример №2.1**
    
- Уязвимость: Server-Side Template Injection (SSTI).
- Строки:
    ```
    output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
    ```
- Последствия: Эксплуатация SSTI может позволить злоумышленнику выполнить произвольный код на сервере, что может привести к утечке данных, повышению привилегий или даже к полному контролю над сервером.
- Решение: Для решения этой уязвимости надо использовать параметризованные шаблоны, где переменные передаются в шаблон через аргументы render(), а не через конкатенацию строк.
    ```
    template = Environment(autoescape=select_autoescape()).from_string('Hello {{ name }}! Your age is {{ age }}.')
    output = template.render(name=name, age=age)
    ```
    **Environment** создает специальное пространство, где будут помещаться наши параметры.

    **autoescape:** Этот параметр помогает предотвратить внедрение потенциально вредоносного кода

    **select_autoescape():** Это функция, которая возвращает значение для параметра autoescape.

    Функция **render()** будет заполнять пустое место для имени **({{ name }})**, которое дал нам пользователь, и для возраста **({{ age }})**

    Также исправим строку 8, во избежание других уязвимостей
    ```
    name = request.values.get('name', 'guest')
    ```
    Теперь, если пользователь не сообщит нам свое имя, мы зачтем его как гостя.


**Пример №2.2**
- Уязвимость: Command Injection.
- Строки:
  ```
  cmd = 'nslookup ' + hostname
  ```
- Последствия: Эксплуатация этой уязвимости может позволить злоумышленнику выполнять произвольные команды на сервере.
- Решение:
  ```
  from flask import Flask, request
  import socket
  app = Flask(__name__)

  @app.route("/dns")
  def dns_lookup():
    hostname = request.values.get('hostname', 'example.com') 
    try:
        ip_address = socket.gethostbyname(hostname)
        return f'The IP address of {hostname} is {ip_address}'
    except socket.gaierror:
        return "Failed to get IP address"
    
  if __name__ == "__main__":
  app.run(debug=True)
  ```

В исправленной версии кода инъекция невозможна, поскольку **socket.gethostbyname(hostname)** не исполняет внешние команды и работает строго в рамках API Python для сетевых операций, обеспечивая безопасность выполнения DNS-запросов без риска внедрения вредоносных команд.

## 3. Моделировани угроз

- Для данного сервиса существует несколько потенциальных проблем безопасности:
  DDoS-Атаки через общедоступные точки входа.
  Если доступ к базе данных не защищен должным образом, то возможны SQL-инъекции и утечка данных клиентов.<br/>
	На данной диаграмме отсутствует механизм мониторинга для отслеживания подозрительной активности.
- Данные проблемы могут привести к сбою всего сервиса, а также к компрометации базы данных, что приведет к утечке данных пользователей.
- Решение проблем: На диаграмме блок Auth, отвечающий за аутентификацию и авторизацию, находится под слоем Backend application. Это может быть логично, если аутентификация реализована как часть микросервисов.<br/>     Однако, если Auth — это отдельный сервис, который должен предшествовать любым операциям внутри backend, его стоит расположить на уровне, предшествующем взаимодействию с микросервисами, для улучшения безопасности и централизации контроля доступа.

  Введение системы обнаружения и предотвращения вторжений, а также журналирования.<br/>
  Также следует разбить backend application на отдельные микросервисы, каждый из которых развернут в изолированной среде для минимизации атак. 
- Какие методы шифрования используются при передаче данных? <br/>
Какие инструменты используются для контроля доступа к компонентам и данным? <br/>
Какие процедуры резервного копирования и восстановления данных применяются?

