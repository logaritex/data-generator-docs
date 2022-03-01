# Usage

Let’s assume that our data application needs to process datasets like this:

```json title="user.json"
{
    "id": "200-66-1234", 
    "sendAt": 1645703265000,
    "fullName": "Laurent Broudoux", 
    "email": "laurent@logaritex.com", 
    "age": 41 
}
```

How to generate more, random instances that resemble the same data model?

#### Avro Schema

First define an [Avro Schema](https://avro.apache.org/docs/1.11.0/spec.html) to model the data structure in the snipped:

=== "YAML"

    ```yaml title="user.yaml"
        namespace: io.simple.clickstream
        type: record
        name: User
        fields:
        - name: id
          type: string
        - name: sendAt
          type:
            type: long
            logicalType: timestamp-millis
        - name: fullName
          type: string
        - name: email
          type: string
        - name: age
          type: int
    ```

=== "JSON"
    
    ```json title="user.json"
     { 
        "namespace": "io.simple.clickstream",
        "type": "record",
        "name": "User",
        "fields": [
            { "name": "id", "type": "string" },
            { "name": "sendAt",
              "type": { "type": "long", "logicalType": "timestamp-millis" }},
            { "name": "fullName", "type": "string" },
            { "name": "email", "type": "string" },
            { "name": "age", "type": "int"}
        ]
    }
    ```

The `DataGenerator` supports `YAML` and `JSON` schema formats.

Now let’s use the schema with the generator and produce few random User instances:

```java
DataUtil.print(
  new DataGenerator(DataUtil.uriToSchema("file:/user.yaml"), 3));
```

you should see something like:

```json
{ 
  "id": "osmmmvlevdsyqygxugaskisncdxo", 
  "sendAt": 2677582661775625024, 
  "fullName": "nhjfdferq", 
  "email": "pwsefuxlgdpho", 
  "age": 313083451
}{
  "id": "hvkebrrghfaypil", 
  "sendAt": -1024915532875297456, 
  "fullName": "ycwixfmtfnelwjmcvrcevsifjwbpmvajjsb", 
  "email": "ceul", 
  "age": 2062740045
}{
  "id": "gppjlulkgyrwdtjohbkyvsmxbou", 
  "sendAt": 4450438112042238783, 
  "fullName": "fkq", 
  "email": "xrddnqbssayshglvsogvthwnpyvosfpxmedvchd", 
  "age": 1128364621
}
```

Though valid and compliant with the schema, the content is neither realistic nor readable. 
We can see in the result negative timestamps and age of 1128364621!

#### Feild Content Expressions

To improve on it we can add type hints to the schema’s [field doc](https://avro.apache.org/docs/1.11.0/spec.html#schema_record) attributes:

```yaml
namespace: io.simple.clickstream
type: record
name: User
fields:
 - name: id
   type: string
   doc: "#{id_number.valid}" 
 - name: sendAt
   type:
     type: long
     logicalType: timestamp-millis
   doc: "[[T(System).currentTimeMillis()]]" 
 - name: fullName
   type: string
   doc: "#{name.fullName}"
 - name: email
   type: string
   doc: "#{internet.emailAddress}"
 - name: age
   type: int
   doc: "#{number.number_between '8','80'}"
```

We can use the `doc` attributes, a common place for Schema documentation, to add metadata about the content to be generated.
Metadata such as [Data Faker](https://www.datafaker.net/) and/or [SpEL](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions) expressions, used to fine tune the generated Datasets for a particular data model or use case.

The extensive list of Faker [Providers](https://www.datafaker.net/providers/) helps to model data for many different domains and different use cases. 
The SpEL provides additional capability for adding conditions, aggregating expressions or even calling Java code directly, when generating field values.

With the annotated schema we can generate realistic user data:

```json
{
  "id": "271-55-3647", 
  "sendAt": 1645480217154, 
  "fullName": "Taunya Kautzer", 
  "email": "mervin.wolf@gmail.com", 
  "age": 66
}{
  "id": "194-48-7155", 
  "sendAt": 1645480217257, 
  "fullName": "Miss Jen Quitzon", 
  "email": "ezekiel.brakus@yahoo.com", 
  "age": 22
}{
  "id": "337-31-9559", 
  "sendAt": 1645480217262, 
  "fullName": "Mr. Shannon Padberg", 
  "email": "derrick.bartoletti@gmail.com", 
  "age": 11
}
```

It looks nicer, but we can see that the email addresses are not related to the user names they belong to!

It is common for some fields in a record to depend on each other. 
How to express such dependencies? 
In particular, can we derive teh `email` address from the `fullName` value?

#### Inter-field dependencies

To do this we can define named expressions in the Record doc attribute. For example the `name=#{name.fullName}` will compute a realistic [full-name](https://s01.oss.sonatype.org/service/local/repositories/releases/archive/net/datafaker/datafaker/1.1.0/datafaker-1.1.0-javadoc.jar/!/net/datafaker/Name.html#fullName()) and assigns it to a [SpEL context variable](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions-ref-variables) called `name`. Once computed this variable can be used inside the field expressions:

```yaml hl_lines="4" linenums="1"
namespace: io.simple.clickstream
type: record
name: User
doc: "name=#{name.fullName}"
fields:
 - name: id
   type: string
   doc: "#{id_number.valid}"
 - name: sendAt
   type:
     type: long
     logicalType: timestamp-millis
   doc: "[[T(System).currentTimeMillis()]]"
 - name: fullName
   type: string
   doc: "[[#name]]" # (1)
 - name: email
   type: string
   doc: "#{internet.emailAddress 
         '[[#name.toLowerCase().replaceAll(\"\\s+\", \".\")]]'}" # (2)
 - name: age
   type: int
   doc: "#{number.number_between '8','80'}"
```

1. Reusing the computed `#name` as a `fullName` field value.
2. Compute the email domain with faker `internet.emailAddress` expression. Derive the email name form the `#name` variable.

Now the same name variable (computed once at the record level) is used directly as `fullName` and also as a part of the expression computing the `email` address.  

!!! tip
    multiple variables can be assigned separated by the `;` character.

Now the user email is derived from user's full name:

```json
{
  "id": "828-71-2990", 
  "sendAt": 1645480217314, 
  "fullName": "Starla Torphy", 
  "email": "starla.torphy@gmail.com", 
  "age": 11
}{
  "id": "341-19-7496", 
  "sendAt": 1645480217332, 
  "fullName": "Ilana Jones Sr", 
  "email": "ilana.jones.sr@yahoo.com", 
  "age": 36
}{
 "id": "460-29-1546", 
 "sendAt": 1645480217337, 
 "fullName": "Antwan Farrell", 
 "email": "antwan.farrell@gmail.com", 
 "age": 31
}
```

If we generate enough instances, it is likely that some user IDs will start to repeat. But although the IDs are the same the rest of the content is different.

To demonstrate this behavior let’s temporarily change our `user.id` expression from `#{id_number.valid}` to `#{options.option '100-00-0000','200-00-0000'}` - e.g. forcing it to choose between one of two fixed IDs:

```json
{"id": "100-00-0000", "sendAt": 1645518230007, "fullName": "Jina Ryan", "email": "jina.ryan@gmail.com", "age": 10}
{"id": "200-00-0000", "sendAt": 1645518230025, "fullName": "Shaniqua Kris PhD", "email": "shaniqua.kris.phd@yahoo.com", "age": 20}
{"id": "100-00-0000", "sendAt": 1645518230030, "fullName": "Tatum O'Connell", "email": "#{internet.emailAddress 'tatum.o'connell'}", "age": 48}
{"id": "200-00-0000", "sendAt": 1645518230032, "fullName": "Effie Rempel", "email": "effie.rempel@hotmail.com", "age": 36}
{"id": "100-00-0000", "sendAt": 1645518230035, "fullName": "Celestina Mertz", "email": "celestina.mertz@yahoo.com", "age": 52}
```

How can we enforce an uniqueness constraint, so that if a selected field value for two instances is the same then those instance contents will be identical as well?

#### Instance Uniqueness

The `unique_on=my-field-name` record level variable helps to set a field name as an instance identifier.  Here is how we can configure unique User instances by user IDs:

```yaml hl_lines="4"
namespace: io.simple.clickstream
type: record
name: User
doc: "unique_on=id;name=#{name.fullName}"
fields:
 - name: id
   type: string
   doc: "#{options.option '100-00-0000','200-00-0000'}"
   #    doc: "#{id_number.valid}"
 - name: sendAt
   type:
     type: long
     logicalType: timestamp-millis
   doc: "[[T(System).currentTimeMillis()]]"
 - name: fullName
   type: string
   doc: "[[#name]]"
 - name: email
   type: string
   doc: "#{internet.emailAddress '[[#name.toLowerCase().replaceAll(\"\\s+\", \".\")]]'}"
 - name: age
   type: int
   doc: "#{number.number_between '8','80'}"
```

Now if the ids are identical the rest of the records are identical as well:

```json
{"id": "100-00-0000", "sendAt": 1645526597895, "fullName": "Nathaniel Sawayn I", "email": "nathaniel.sawayn.i@hotmail.com", "age": 49}
{"id": "100-00-0000", "sendAt": 1645526597895, "fullName": "Nathaniel Sawayn I", "email": "nathaniel.sawayn.i@hotmail.com", "age": 49}
{"id": "200-00-0000", "sendAt": 1645526597923, "fullName": "Letha Bauch", "email": "letha.bauch@yahoo.com", "age": 21}
{"id": "200-00-0000", "sendAt": 1645526597923, "fullName": "Letha Bauch", "email": "letha.bauch@yahoo.com", "age": 21}
{"id": "100-00-0000", "sendAt": 1645526597895, "fullName": "Nathaniel Sawayn I", "email": "nathaniel.sawayn.i@hotmail.com", "age": 49}
```

!!! note
    For the uniqueness constraint, current implementation retains the generated records in memory which could lead to OOM. (TODO: to implement a ring buffer and/or external metadatastore strategies to mitigate this issue).

#### Shared Field Values

Let's expand our use case a bit to model a clickstream scenario. The existing User schema represents the registered users in our website. In addition add the Click schema to represent the clickstream events generated by the (registered) users when they browse the website. 

Here is a sample Click schema:

```yaml title="click.yaml"
click.yaml
namespace: io.simple.clickstream
type: record
name: Click
fields:
 - name: user_id
   type: string
   doc: "#{id_number.valid}"
 - name: page
   type: int
   doc: "#{number.number_between '1','100000'}"
 - name: action
   type: string
   doc: "#{options.option 'vitrine_nav','checkout','product_detail','products','selection','cart'}"
 - name: user_agent
   type: string
   doc: "#{internet.userAgentAny}"
```

The `click.user_id` in the clickstream corresponds to the `user.id` in the User dataset. The `page` field stands for the web page number and the `action` stands for the action performed on that page. The `userAgent` provides information about the user's browser.

Lets generate few instances for both schemas: 

```java
DataUtil.print(new DataGenerator(
           DataUtil.uriToSchema("file:/user.yaml"), 3));

DataUtil.print(new DataGenerator(
           DataUtil.uriToSchema("file:/click.yaml"), 3));
```

!!! note
    Mind that those generators can be run concurrently in parallel!

The user output would look something like this:

```json
{"id": "200-00-0000", "sendAt": 1645535217424, "fullName": "Annmarie Mayer", "email": "annmarie.mayer@yahoo.com", "age": 52}
{"id": "200-00-0000", "sendAt": 1645535217424, "fullName": "Annmarie Mayer", "email": "annmarie.mayer@yahoo.com", "age": 52}
{"id": "100-00-0000", "sendAt": 1645535217436, "fullName": "Gerald Predovic", "email": "gerald.predovic@yahoo.com", "age": 22}
```

and the clicks:

```json
{"user_id": "124-36-2522", "page": 34319, "action": "checkout", "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 12_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0 Mobile/15E148 Safari/604.1"}
{"user_id": "235-28-9926", "page": 67317, "action": "checkout", "user_agent": "Mozilla/5.0 (compatible; Linux x86_64; Mail.RU_Bot/2.0; +http://go.mail.ru/help/robots)"}
{"user_id": "008-06-4580", "page": 90337, "action": "product_detail", "user_agent": "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36"}
```

Datasets look fine except for one flow! If you remember, the user-id in the Click dataset should match existing user IDs in the User dataset! But in the generated content the click#user_id is random and not related to the User dataset. This will make it very hard to use those datasets for clickstream analysis. The systems that process such clickstream data are likely to try to join the data by user ID and will fail to get any meaningful results.

So can we share field values between instances of different datasets? 

The [SharedFieldValuesContext](https://github.com/vmware-tanzu/streaming-runtimes/blob/main/data-generator-parent/data-generator/src/main/java/com/tanzu/streaming/runtime/data/generator/context/SharedFieldValuesContext.java) class is designed to help us with this. It provides a thread-safe field exchange context and can be used across multiple DataGenerator Schema types:

```java
SharedFieldValuesContext sharedFieldValuesContext = 
               new SharedFieldValuesContext(new Random());

DataUtil.print(new DataGenerator(
         DataUtil.uriToSchema("file:/user.yaml"), 3,
         sharedFieldValuesContext));

DataUtil.print(new DataGenerator(
         DataUtil.uriToSchema("file:/click.yaml"), 3,
         sharedFieldValuesContext));
```

Then we can use the `to_share=id` instruction to let our User schema keep the generated user ids in the shared field context:

```yaml hl_lines="4"
namespace: io.simple.clickstream
type: record
name: User
doc: "unique_on=id;name=#{name.fullName};to_share=id"
fields:
 - name: id
   type: string
   doc: "#{options.option '100-00-0000','200-00-0000'}"
#    doc: "#{id_number.valid}"
 - name: sendAt
   type:
     type: long
     logicalType: timestamp-millis
   doc: "[[T(System).currentTimeMillis()]]"
 - name: fullName
   type: string
   doc: "[[#name]]"
 - name: email
   type: string
   doc: "#{internet.emailAddress '[[#name.toLowerCase().replaceAll(\"\\s+\", \".\")]]'}"
 - name: age
   type: int
   doc: "#{number.number_between '8','80'}"
```

On the receiving side we can access the collected user ids values with the help of SpEL expression like `[[#shared.field(‘<schema-name.field-name>’)]]`.
Lets leverage it in the Click Schema:

```yaml hl_lines="7"
namespace: io.simple.clickstream
type: record
name: Click
fields:
 - name: user_id
   type: string
   doc: "[[#shared.field('user.id')?:'666-66-6666']]"
 - name: page
   type: long
   doc: "#{number.number_between '1','100000'}"
 - name: action
   type: string
   doc: "#{options.option 'vitrine_nav','checkout','product_detail','products','cart'}"
 - name: user_agent
   type: string
   doc: "#{internet.userAgentAny}"
```

The `[[#shared.field(‘user.id’)]]` retrieves a random value from the shared field context for the specified `<schema>.<field-name> name`.

If no data is found the expression will return NULL which in turn will result in null instances! To prevent null responses you can use the Elvis expression to set a default value on missing data.

If we re-run the generators for both data sources we will see that the `click#user_ids` are drawn from the pool of values generated for the `user#id`:

```json title="users"
{"id": "200-00-0000", "sendAt": 1645540862314, "fullName": "Ronnie Shields", "email": "ronnie.shields@hotmail.com", "age": 52}
{"id": "200-00-0000", "sendAt": 1645540862314, "fullName": "Ronnie Shields", "email": "ronnie.shields@hotmail.com", "age": 52}
{"id": "100-00-0000", "sendAt": 1645540862324, "fullName": "Tijuana Watsica", "email": "tijuana.watsica@hotmail.com", "age": 15}
```

```json title="clickstream"
{"user_id": "100-00-0000", "page": 819, "action": "selection", "user_agent": "Mozilla/4.0 (compatible; MSIE 6.0; AOL 9.0; Windows NT 5.1; .NET CLR 1.1.4322)"}
{"user_id": "100-00-0000", "page": 99145, "action": "cart", "user_agent": "Mozilla/5.0 (Windows NT 6.2; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36"}
{"user_id": "200-00-0000", "page": 21073, "action": "selection", "user_agent": "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:18.0) Gecko/20100101 Firefox/18.0"}
```

The [SharedFieldValuesContext](https://github.com/vmware-tanzu/streaming-runtimes/blob/main/data-generator-parent/data-generator/src/main/java/com/tanzu/streaming/runtime/data/generator/context/SharedFieldValuesContext.java) is thread safe and dynamic.

#### Map Fields

Avro Maps allows to define dynamic, nested key/value structures where the key is always of type string while the value has its own schema. 

Let’s start with a simple PetShop Avro schema:

```yaml
namespace: my.petshop
type: record
name: PetShop
fields:
 - name: pet_shop
   type: string
 - name: cats
   type:
     type: map
     values: string
```

and generate few random instances

```json
{
  "pet_shop": "wojddtswl", 
   "cats": {
      "emduhsbcbskmedysmnj": "vnkghckskmmgvatmghraarfnmjpoekttx",     
      "shlrcvatwdkwbbydmnekcvsfuheaqc": "deds",
      "fwiw": "inpowwtbqfhxctcsdx"
  }
}
{
  "pet_shop": "ysygvscckfuxcxuqjmiwajtambmvbrvngtpiu", 
  "cats": {"gx": "rshmm"}
}
{ 
  "pet_shop": "qnryaurkhyhiyxptdcdhtlbasenknlwghon",
  "cats": {"akrejxplgrxdmvqjsrhxbvcqnpkhkkgg": "ydacyjviowlvgklbkqg"}
}
```

valid data instances but not realistic.
Now let's add few hints:

```yaml hl_lines="12"
namespace: my.petshop
type: record
name: PetShop
fields:
 - name: pet_shop
   type: string
   doc: "[[#faker.company().name()]]"
 - name: cats
   type:
     type: map
     values: string
   doc: "key=#{cat.name};value=#{cat.breed};length=#{number.number_between '0','10'}"
```

For the `pet_shop` a regular Faker expression is used to generate random company names.
The `cats` field is more interesting as we define specific `length`, `key` and `value` variables.
The `length` is resolved first and used to determine the number map’s key/value elements.
The `key` and the `value` are resolved every time a new map element is generated.
The updated result looks like this:

```json
{
   "pet_shop": "Yost, Green and Rohan", 
   "cats": {
      "Poppy": "Cymric, or Manx Longhair", 
      "Max": "Bombay", 
      "Coco": "Pixie-bob", 
      "Shadow": "Siamese", 
      "Chloe": "Balinese", 
      "Tiger": "Chantilly-Tiffany", 
      "Smudge": "Exotic Shorthair"
   }
}
{
   "pet_shop": "Jones-Wintheiser", 
   "cats": {
      "Poppy": "Peterbald", 
      "Shadow": "Oriental Bicolor", 
      "Milo": "Thai", 
      "Chloe": "Persian (Traditional Persian Cat)", 
      "Millie": "Chartreux"
   }
}
{
   "pet_shop": "Kovacek, Gaylord and Ankunding", 
   "cats": {      
      "Milo": "Oriental Longhair", 
      "Lucy": "Egyptian Mau", 
      "Simba": "Pixie-bob"
   }
}
```

!!! tip
    if you use the Faker expression `#{company.name}` in ⅓ of the cases that run into internal Faker issues rendering null records. The save option is to use the Faker Java API vis the SpEL like this `[[#faker.company().name()]]` instead