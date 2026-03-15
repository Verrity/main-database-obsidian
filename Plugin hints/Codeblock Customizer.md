---


---
По умолчанию все блоки сворачиваются, чтобы этого не было нужно
1. не прописывать `title`
2. ставить `unfold`

Для заголовка можно использовать `title="..." или file:...`, экранировать `\"`
```cpp fold nowrap title="Simple collapsible syntax highlighted block"
int main() {
	return 0;
	// PRESS THE WRAPPING BUTTON       dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
}
```
```cpp file:\"test.cpp\"
int main() {
	return 0;
}
```



Группировка блоков
```cpp title="Файл 1" group:test
int main() {
	int a = 5;
	return 0;
}
```

```cpp title="Файл 2" group:test
int main() {
	bool b = false;
	return 0;
}
```






