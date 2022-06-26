# hwloc库使用调研

## 专业术语

Object：

Processing Unit (PU)：

Package：

NUMA Node：

Memory-side Cache：


## 源码分析

```c++
typedef enum {
 #define HWLOC_OBJ_TYPE_MIN HWLOC_OBJ_MACHINE /* Sentinel value */
   HWLOC_OBJ_MACHINE,    
   HWLOC_OBJ_PACKAGE,    
   HWLOC_OBJ_CORE,       
   HWLOC_OBJ_PU,         
   HWLOC_OBJ_L1CACHE,    
   HWLOC_OBJ_L2CACHE,    
   HWLOC_OBJ_L3CACHE,    
   HWLOC_OBJ_L4CACHE,    
   HWLOC_OBJ_L5CACHE,    
   HWLOC_OBJ_L1ICACHE,   
   HWLOC_OBJ_L2ICACHE,   
   HWLOC_OBJ_L3ICACHE,   
   HWLOC_OBJ_GROUP,      
   HWLOC_OBJ_NUMANODE,   
   HWLOC_OBJ_BRIDGE,     
   HWLOC_OBJ_PCI_DEVICE, 
   HWLOC_OBJ_OS_DEVICE,  
   HWLOC_OBJ_MISC,       
   HWLOC_OBJ_MEMCACHE,   
   HWLOC_OBJ_DIE,        
   HWLOC_OBJ_TYPE_MAX    
 } hwloc_obj_type_t;

```