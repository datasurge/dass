# jupyterlab 接口文档(Add_model.py)

开发人员：王少松

创建时间：2018-08-09

***

## 接口简介

将数据增加到Mysql数据库里面的models表和user_model_type表 --- （Add_model.py）

返回前端json数据---模型分类和基础信息   ----  （Type_handler.py）

### 接口详情

***

### 请求地址

'/lab/api/addmodel'  ----（Add_model.py）

'/lab/api/type'    ---(Type_handler.py)

### 请求类型

get 、post

### 请求参数

| 参数名     | 类型   | 必填 | 描述           | 默认值 | 参考值          |
| ---------- | ------ | ---- | -------------- | ------ | --------------- |
| Model_Name | 整型   | 是   | 模型名字       | -      | model测试1      |
| Code       | 字符串 | 是   | 模型代码       | -      | print("520520") |
| Permission | 整型   | 是   | 权限           | -      | 1               |
| Desc       | 字符串 | 是   | 对该模型的描述 | -      | 输出520520      |
| ctype_id   | 整型   | 是   | 子类模型ID     | -      | 2               |

（Add_model.py）

### 返回json示例

{  

   "type": [  {  

​           "father_id": 1, 

​            "type_name": "type测试1", 

​            "child": [ 

​		{                     

​			"cid": 3,                    

​			 "cname": "type测试3",                     

​			"mdesc": [

​	                         {                             

​					"mid": 1,                            

​					 "mname": "model测试1"   

​			 },

​                         { 

​			"mid": 5,                            

​			 "mname": "model测试5"                         

​			}                    

​		 ]                

​	 },                 

​	{ 

​	  "cid": 5,                    

​	 "cname": "type测试5",                     

​	"mdesc": [  

​		{                             

​			"mid": 3,                             

​			"mname": "model测试3"

​		}                     

​	]                

 }            

 ]         

},         

{            

 "father_id": 2,             

"type_name": "type测试2",             

"child": [                 

​	{                     

​		"cid": 4,                     

​		"cname": "type测试4",                     

​		"mdesc": [                        

​			 {                            

​				 "mid": 2,                            

​				 "mname": "model测试2"                         

​			},                        

​			 {                            

​				 "mid": 4,                            

​				 "mname": "model测试4"                         

​			}                     

​				]                 

​		},                

​		 {                     

​			"cid": 6,                    

​			 "cname": "type测试6",                    

​			 "mdesc": []                

​		 }            

​		 ]         

​	}     ]

 } 

### 返回数据

'/lab/api/addmodel'

​	get方法：return model（对象）

​	返回前端字符串（保存成功或失败）

'/lab/api/type'

​	get方法：返回前端json数据。

### 备注说明

user_id 目前还无法获取，临时随即取出一整数

### 修改日志

无

