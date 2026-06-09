---
layout: post
title: "유니티 게임에서 dll injection 후 게임 내부 요소 조절"
summary: "유니티 게임에서 dll injection 후 게임 내부 요소 조절"
author: eveheeero
date: '2026-06-09 14:36:00 +0900'
category: ['reversing', 'learning']
tags: reversing
thumbnail: /assets/img/posts/2026-06-09-unity-hooking-0.png
keywords: reversing
usemathjax: false
permalink: /blog/unity-hooking/
---

내부 파일입니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-0.png){: style="max-width: 100%; height: auto;"}

die로 분석해보니 유니티인데도 관련 파일들이 전부 CPP로 되어있다고 표시됩니다. dnspy도 동작하지 않습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-1.png){: style="max-width: 100%; height: auto;"}

![_](/assets/img/posts/2026-06-09-unity-hooking-2.png){: style="max-width: 100%; height: auto;"}

내부 파일을 살펴보니 `il2cpp` 라는 키워드를 찾아볼 수 있었고, 살펴보니 닷넷 머신이었습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-3.png){: style="max-width: 100%; height: auto;"}

`Il2CppDumper`를 찾아서 원본 dll파일을 일부 뽑아낼 수 있었습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-4.png){: style="max-width: 100%; height: auto;"}

![_](/assets/img/posts/2026-06-09-unity-hooking-5.png){: style="max-width: 100%; height: auto;"}

![_](/assets/img/posts/2026-06-09-unity-hooking-6.png){: style="max-width: 100%; height: auto;"}

디버거로 살펴보니 `gameassembly.dll`에서 il2cpp 내부 함수들을 호출해서 닷넷 어셈블리를 호출하는것을 볼 수 있습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-7.png){: style="max-width: 100%; height: auto;"}

위 내용들을 살펴봐, 특정 메소드를 호출하는 함수는 `il2cpp_runtime_invoke`라는것을 알아냈습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-8.png){: style="max-width: 100%; height: auto;"}

다음과 같은 코드를 작성해 ShowFlag를 호출하였습니다. 호출은 성공하였지만 인스턴스가 달라서인지 아무것도 나오지 않았었습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-9.png){: style="max-width: 100%; height: auto;"}

닷넷 어셈블리 내용을 보니 플래그는 게임 내부에 표시되는 것으로 보이며, static필드에 초기화 된 인스턴스가 있는 것이 확인되었습니다.

---

자세히 찾아보니 `TextMeshProUGUI`는 내부에 원본 문자열을 가지고 있었습니다.

`instance`를 가져온 후, `flagText`를 가져온 후 원본 데이터를 가져올 수 있습니다. `il2cpp_string_chars`

---

자세히 살펴보니 TextMeshProUGUI내부 text는 `TMP_Text`에 의해 정의되어 있었는데, getter가 없으며 setter만 있어서, 실제로 필드가 있는 것이 아닌 것으로 보였습니다.

대신 불러온 인스턴스를 이용해 ShowFlag를 표시해보니 다음과 같이 텍스트가 표시되었습니다.

![_](/assets/img/posts/2026-06-09-unity-hooking-10.png){: style="max-width: 100%; height: auto;"}

`OnButtonPressed` 및 `CalculateSHA256`함수 진행 중 플래그가 완성되는 듯합니다.

직접 움직여서 버튼을 눌러야 할듯합니다

가속도등의 데이터를 조작해보기 위해 게임 오브젝트 인스턴스를 찾아보았으나, dll을 통해 해당 오브젝트를 가져오는 방법을 찾지 못했습니다

`FindObjectsOfType`등의 api가 없었음, Object 내부 FindObjectFromInstanceID를 통해 다양한 오브젝트를 가져올 수 있는것으로 보이나, 오브젝트 아이디는 해시값으로 보였음

```c++
void main() {
  typedef  const void* (*il2cpp_domain_get_t)();
  typedef  const void* (*il2cpp_domain_assembly_open_t)(const void*, const char*);
  typedef const void* (*il2cpp_assembly_get_image_t) (const void*);
  typedef const void* (*il2cpp_class_from_name_t) (const void*, const char*, const char*);
  typedef const void* (*il2cpp_object_new_t) (const void*);
  typedef const void* (*il2cpp_class_get_method_from_name_t) (const void*, const char*, int);
  typedef const void* (*il2cpp_runtime_invoke_t) (const void*, const void*, void**, void**);
  HMODULE hIl2cpp = GetModuleHandleA("gameassembly.dll");
  il2cpp_domain_get_t il2cpp_domain_get = (il2cpp_domain_get_t) GetProcAddress(hIl2cpp, "il2cpp_domain_get");
  il2cpp_domain_assembly_open_t il2cpp_domain_assembly_open = (il2cpp_domain_assembly_open_t)GetProcAddress(hIl2cpp, "il2cpp_domain_assembly_open");
  const void* il2cppDomain = il2cpp_domain_get();
  const void* il2cppMainAssembly = il2cpp_domain_assembly_open(il2cppDomain,"Assembly-CSharp.dll");
  OutputDebugStringA("main assembly open");
  il2cpp_assembly_get_image_t il2cpp_assembly_get_image = (il2cpp_assembly_get_image_t)GetProcAddress(hIl2cpp, "il2cpp_assembly_get_image");
  const void* il2cppMainImage= il2cpp_assembly_get_image(il2cppMainAssembly);
  OutputDebugStringA("main image open");
  il2cpp_class_from_name_t il2cpp_class_from_name = (il2cpp_class_from_name_t)GetProcAddress(hIl2cpp, "il2cpp_class_from_name");
  const void* il2cppClass = il2cpp_class_from_name(il2cppMainImage, "", "GameManager");
  OutputDebugStringA("gamemanager class loaded");
  typedef const void* (*il2cpp_class_get_field_from_name_t) (const void*, const char*);
  typedef void (*il2cpp_field_static_get_value_t) (const void*, void*);
  il2cpp_class_get_field_from_name_t il2cpp_class_get_field_from_name = (il2cpp_class_get_field_from_name_t)GetProcAddress(hIl2cpp, "il2cpp_class_get_field_from_name");
  const void* il2cppClassField= il2cpp_class_get_field_from_name(il2cppClass, "instance");
  OutputDebugStringA("gamemanager class field loaded");
  void* il2cppClassInstance=nullptr;
  il2cpp_field_static_get_value_t il2cpp_field_static_get_value = (il2cpp_field_static_get_value_t)GetProcAddress(hIl2cpp, "il2cpp_field_static_get_value");
  il2cpp_field_static_get_value(il2cppClassField, &il2cppClassInstance);
  OutputDebugStringA("gamemanager class instance loaded");

  il2cpp_class_get_method_from_name_t il2cpp_class_get_method_from_name = (il2cpp_class_get_method_from_name_t)GetProcAddress(hIl2cpp, "il2cpp_class_get_method_from_name");
  const void* rShowFlag = il2cpp_class_get_method_from_name(il2cppClass, "ShowFlag", 0);
  OutputDebugStringA("GameManager::ShowFlag readed");
  il2cpp_runtime_invoke_t il2cpp_runtime_invoke = (il2cpp_runtime_invoke_t)GetProcAddress(hIl2cpp, "il2cpp_runtime_invoke");
  void* params[] = { 0 };
  il2cpp_runtime_invoke(rShowFlag, il2cppClassInstance, params, 0);
  OutputDebugStringA("method call success");
}

```

---

의도된 풀이는 EasyHook 라이브러리를 이용해, 캐릭터 이동 함수 시작부분을 후킹하여 메소드를 호출하는 것이라고 합니다
