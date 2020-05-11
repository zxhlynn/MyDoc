# NSCoding 和 Codable

自定义对象的保存是程序中的一个重要功能，ObjC中提供了 `NSCoding/NSSecureCoding` 协议供对象实现以满足 `archive` 和 `uncharive` 的要求。

## NSCoding

`NSCoding` 在ObjC中就已经存在，在Swift的定义和使用和ObjC中一样，它的主要功能是归档对象，便于存储。实现 `NSCoding` 协议必须实现两个函数：

```
required init(coder aDecoder: NSCoder) {
    name = aDecoder.decodeObject(forKey: "name") as? String
    self.init(name)

}

func encode(with aCoder: NSCoder) {
    aCoder.encode(name, forKey: "name")
}
```

只有实现了NSCoding协议的类对象，才可以使用 NSKeyedArchiver.archivedData 归档成文件然后进行存储。注意，只有class才能实现该协议。

## Codable

Codable是Swift中独有的协议，使用于各种自定义的类型(class和struct都可以实现该协议)。它的主要作用是使自定义的数据可以表示成各种外部数据表现形式，如常见的JSON。当我们需要向服务器发送数据，存储成文件时都需要自定义数据类型可以被编码和解码。

Codable的定义：

```
public typealias Codable = Decodable & Encodable
```

可以发现，Swift中把编码和解码分别定义成单独的协议，也就是说根据实际情况，可以只实现 `Encodable` 或 `Decodable。`
常见的 `String` `，Int` ， `Array` (如果元素实现了 `Codable` )等结构体和 `Foundation` 标准库中的 `Date` ， `Data` 等类型都实现了 `Codable` 。如果自定义类型的属性实现了 `Codable` 的话，那自定义的类型也就默认实现了 `Codable` ，唯一需要做的就是声明实现 `Codable` ，如：

```
struct MyPerson: Codable {
  var name: String
  var age: Int
}
```

那对形如下面结构的json对象可以进行编码/解码:

```
[ { "name": "Sun", "age": 20 }, { "name": "John", "age": 19 } ]

//decode 
let jsonData = jsonString.data(using: .utf8)! 
let persons = try JSONDecoder().decode([MyPerson].self, from: jsonData) 
//encode 
var encoder = JSONEncoder() 
encoder.outputFormatting = .prettyPrinted 
let encodeData = try encoder.encode(persons)
```

### key名字不一致

有时候json schema中的key和定义的struct中的属性名不一致或者为了避免和关键字重名，Swift中引入了CodingKey协议，示例如下：

```
//假设JSON数据变为：
[
  {
      "student_name": "Sun",
      "age": 20
  },
  {
      "student_name": "John",
      "age": 19
  }
]
//相应的增加枚举类型Keys，其raw value为对应的key，如果一致可以省略
struct MyPerson: Codable {
  var name: String
  var age: Int
  private enum CodingKeys: String,CodingKey{
    case name = "student_name"
    case age
  }
}
```
注意，如果没有实现Codable定义的函数时，enum名字必须为 CodingKeys

### 属性深度不一致

有时候收到的json数据如：

```
[ 
  { 
    "name":"Suny Island", 
    "foundingYear":1976, 
    "latitude": 102.9870, 
    "longitude": 32.9870 
    }, 
    { 
      "name":"Green park", 
      "foundingYear":2008, 
      "latitude": 32.9870, 
      "longitude": 27.9870 
      } 
]
```

而定义的模型：

```
struct Coordinate: Codable {
  var latitude: Double
  var longitude: Double
}

struct Landmark: Codable {
  var name: String
  var founding: Int
  var location: Coordinate
}
```

这时我们就需要实现Encodable和Decodable协议--当然，可以根据实际情况只实现Decodable。需要添加的部分：

```
private enum CodingKeys:String, CodingKey {
    case name
    case founding = "foundingYear"
    case latitude
    case longitude
}

init(from decoder: Decoder) throws {
    let values = try decoder.container(keyedBy: CodingKeys.self)
    name = try values.decode(String.self, forKey: .name)
    founding = try values.decode(Int.self, forKey: .founding)
    location = try Coordinate(from: decoder)
}
func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    try container.encode(name, forKey: .name)
    try container.encode(founding, forKey: .founding)
    try container.encode(location.latitude, forKey: .latitude)
    try container.encode(location.longitude, forKey: .longitude)
}
```

若返回的json中某个属性为null，那么在相应的模型中使其变为 optional，并且使用 try? 。

如果是相反的情形，如：

```
[ 
  { 
    "name":"Suny Island", 
    "detail":{ 
      "foundingYear":1976, 
      "latitude": 102.9870, 
      "longitude": 32.9870 
      } 
  }, 
  { 
    "name":"Green park", 
    "detail": { 
      "foundingYear":2008, 
      "latitude": 32.9870, 
      "longitude": 27.9870 
      } 
  } 
]
```

而定义的模型：

```
struct Landmark: Codable {
  var name: String
  var foundingYear: Int
  var latitude: Double
  var longitude: Double
}
```
这种情况下需要用到nestedContainer：

```
private enum CodingKeys:String, CodingKey {
    case name
    case detail

}
private enum DetailKeys: String, CodingKey{
    case foundingYear
    case latitude
    case longitude
}

init(from decoder: Decoder) throws {
    let values = try decoder.container(keyedBy: CodingKeys.self)
    name = try values.decode(String.self, forKey: .name)
    let detailInfo = try values.nestedContainer(keyedBy: DetailKeys.self, forKey: .detail)
    foundingYear = try detailInfo.decode(Int.self, forKey: .foundingYear)
    latitude = try detailInfo.decode(Double.self, forKey: .latitude)
    longitude = try detailInfo.decode(Double.self, forKey: .longitude)
}
func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    try container.encode(name, forKey: .name)
    var detailInfo = container.nestedContainer(keyedBy: DetailKeys.self, forKey: .detail)
    try detailInfo.encode(foundingYear, forKey: .foundingYear)
    try detailInfo.encode(latitude, forKey: .latitude)
    try detailInfo.encode(longitude, forKey: .longitude)
}
```