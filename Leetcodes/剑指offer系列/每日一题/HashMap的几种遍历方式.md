# 一、迭代EntrySet

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();
for(Map.Entry<Integer, Integer> entry : map.entrySet()){
	int key = entry.getKey();
    int value = entry.getValue();
}
```

# 二、单独迭代keys或values

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();
 
for (Integer key : map.keySet()) {
	
    int value = map.get(key);
}
 
for (Integer value : map.values()) {
	System.out.println("Value = " + value);
}
```

# 三、使用Iterator

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();

Iterator<Map.Entry<Integer, Integer>> entries = map.entrySet().iterator();
while (entries.hasNext()) {
	Map.Entry<Integer, Integer> entry = entries.next();
	int key = entry.getKey();
    int value = entry.getValue());
}
```

