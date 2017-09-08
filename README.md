# FixJSONModel
Single Model with multiple KeyMappers for serializing


项目中使用JSONModel解析接口返回时，遇到一个model对应多个接口字段，例如

```
A:
contacts = (
{
contactId = xx;
contactName = "xx";
},
{
contactId = xx;
contactName = "xx";
}
);

B:
contacts = (
{
id = xx;
name = "xxx";
},
{
id = xx;
name = "xx";
}
);
```

对应的model

```
@protocol MSContactModel;

@interface MSContactModel : JSONModel

@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *contactType;
@property (nonatomic, strong) NSString *cellphone;
@property (nonatomic, strong) NSString *telephone;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSString *contactId;

@end

.m

+(JSONKeyMapper *)keyMapper
{
NSMutableDictionary *keyMapper = [NSMutableDictionary dictionary];
[keyMapper setObject:@"id" forKey:@"contactId"];
[keyMapper setObject:@"name" forKey:@"contactName"];
return [[JSONKeyMapper alloc] initWithDictionary:keyMapper];
}
```

这样会造成接口返回字段为name时，不能正确解析。
原因是
JSONModel.m
```
-(BOOL)__importDictionary:(NSDictionary*)dict withKeyMapper:(JSONKeyMapper*)keyMapper validation:(BOOL)validation error:(NSError**)err
{
//loop over the incoming keys and set self's properties
for (JSONModelClassProperty* property in [self __properties__]) {

//convert key name ot model keys, if a mapper is provided
NSString* jsonKeyPath = (keyMapper||globalKeyMapper) ? [self __mapString:property.name withKeyMapper:keyMapper importing:YES] : property.name;//这里会根据keyMapper返回
//JMLog(@"keyPath: %@", jsonKeyPath);

//general check for data type compliance
id jsonValue;
@try {
jsonValue = [dict valueForKeyPath: jsonKeyPath];
}
@catch (NSException *exception) {
jsonValue = dict[jsonKeyPath];
}
```

解决方法
JSONModel.m
```
-(BOOL)__importDictionary:(NSDictionary*)dict withKeyMapper:(JSONKeyMapper*)keyMapper validation:(BOOL)validation error:(NSError**)err
{
//loop over the incoming keys and set self's properties
for (JSONModelClassProperty* property in [self __properties__]) {

//convert key name ot model keys, if a mapper is provided
NSString*jsonKeyPath = nil;
//JMLog(@"keyPath: %@", jsonKeyPath);

//这里先判断是否可以直接解析
if (dict[property.name]) {
jsonKeyPath = property.name;
}else{
jsonKeyPath = (keyMapper) ? [self __mapString:property.name withKeyMapper:keyMapper importing:YES] : property.name;
}
//general check for data type compliance
id jsonValue;
@try {
jsonValue = [dict valueForKeyPath: jsonKeyPath];
}
@catch (NSException *exception) {
jsonValue = dict[jsonKeyPath];
}
```

由于直接修改源码会导致，更新pod的时候会被覆盖，所以创建Category
JSONModel+Track

JSONModel+Track.m
```
...........
if (dict[property.name]) {
jsonKeyPath = property.name;
}else{
jsonKeyPath = (keyMapper) ? [self __mapString:property.name withKeyMapper:keyMapper importing:YES] : property.name;
}
...........
```

[代码 FixJSONModel](https://github.com/name002/FixJSONModel)

参考

[Single Model with multiple KeyMappers for serializing](https://github.com/jsonmodel/jsonmodel/issues/545)
[JSONModel命名中的驼峰(CamelCase)与下划线 (UnderscoreCase)](http://blog.csdn.net/baidu_32469997/article/details/51882917)
