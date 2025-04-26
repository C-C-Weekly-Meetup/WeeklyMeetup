## 개요
최근 웹 애플리케이션 서버의 URL 패턴 매핑 기능을 직접 구현하던 중, 접두사 매칭(Prefix match)에 대한 요구사항을 마주하게 되었다. 예를들어 사용자에게 `/user/test` URL 요청이 오면 `/user/test/*` 와 `/user/*` 중 더 구체적인 패턴, 즉 가장 긴 접두사인 `user/test/*`가 매핑되어야 한다. 

정확히 일치하는 URL을 찾는 것이라면 HashMap 자료 구조를 이용하여 O(1)로 처리할 수 있다. 하지만 와일드 카드("\*")가 포함된 접두사 매칭은 일반적인 해시 기반 탐색만으로 해결하기 어려웠다. 때문에 순차 탐색으로 startsWith()와 longest 변수를 두어 요청 URL과 접두사가 일치하고 가장 긴 패턴이면 갱신하도록 구현하였다. 시간 복잡도는 순차 탐색이므로 O(n) (패턴 수에 비례)이다.
```java
[코드 일부]
// 2. Prefix match (가장 긴 prefix 우선)  
StandardWrapper prefixMatch = null;  
int longest = -1;  
for (Map.Entry<String, StandardWrapper> entry : prefixMappings.entrySet()) {  
    String prefix = entry.getKey();  
    if (uri.startsWith(prefix) && prefix.length() > longest) {  
        prefixMatch = entry.getValue();  
        longest = prefix.length();  
    }  
}
```

첫 번째로 고려한 개선 방향은 **이진 탐색(Binary Search)** 을 적용하여 O(logN)으로 탐색 효율을 높이는 것이였다.
정렬된 URL 패턴 목록이 있다면, 요청 URI에서 접두사를 잘라가며 이진 탐색을 반복해서 가장 긴 매칭 패턴을 찾을 수 있을 것같았다.

예를 들어 URL 패턴이 사전순으로 정렬되어있다고 가정하자.
```bash
/account/*
/account/login/*
/account/profile/*
/book/*
/book/detail/*
...
```
그리고 요청 URL이 `/account/login/page`라고 하면, 접두사 매칭을 위해 우리는 아래와 같이 접두사를 줄여가며 확인해야한다.
```bash
/account/login/page  => 없음
/account/login       => 있음
/account             => 있음
```
하지만 이 구조에는 접두사를 줄여가며 이진 탐색을 여러 번 수행해야 한다. URL 문자열이 L개 이면 O(L logN)시간 복잡도가 발생한다.


## Trie(Prefix Tree)
이진 탐색을 이용한 방식은 아이디어는 괜찮지만, 접두사를 잘라가며 여러번 이진 탐색을 반복해야 한다는 점에서 한계를 가진다.
이러한 문제를 해결하기 위한 대안으로 **문자열을 구조적으로 탐색할 수 있는 자료구조인 Trie가 널리 사용**된다.

Trie(트라이)는 "Re**trie**val Tree"의 줄임말로, 문자열의 접두사 기반으로 정보를 저장하고 검색할 수 있도록 설계된 트리 기반 자료구조이다.

등록된 URL패턴이 아래와 같이 있다고한다면 /기준으로 하나의 노드로 구성한다.
```
[등록 되어있는 URL 패턴들]
/user/*
/user/mypage/*
/user/login/*
/user/login/view*
/user/login/page/edit/*

/home/board/*
/home/news/*
/home/foot/*

```
![Pasted image 20250426231548](https://github.com/user-attachments/assets/acd58661-f4b6-4592-80c6-b9a647b25ffb)


### TrieNode
```java
public class TrieNode {
    // 현재 노드가 하나의 URL segment를 의미
    Map<String, TrieNode> children = new HashMap<>();
    boolean isPatternEnd = false;        // 이 노드에서 패턴이 끝나는지 여부
    Controller value = null;           // URL패턴과 매핑된 정보(Servlet, Controller 등)
}

```

### Trie 삽입 
URL 패턴은 /기준으로 분리한 후, 각 segment를 순차적으로 TrieNode에 삽입한다.(segment: "/"구분되는 문자열)
이미 존재하는 segment는 그대로 재사용하며, 새로운 segment가 등장하면 노드를 생성한다. 패턴의 마지막 노드에는 `isPatternEnd = true`로 표시하고, 매핑된 서블릿 혹은 컨트롤러 객체(`Controller`)를 `value` 필드에 저장한다.

`isPatternEnd == true`는 여기까지 탐색한 경로가 하나의 완성된 URL 패턴이고, 이 요청에 대해 컨트롤러(또는 서블릿)를 반환할 수 있다는 것을 의미한다.
```java
router.insert("/user/login/view", new LoginViewController()); // *생략
router.insert("/user/mypage", new MyPageController()); // *생략
```

삽입의 시간 복잡도는 L(= segment 개수)만큼 소요된다. URL이 `/user/login/page`이면 L=3, 즉 O(L)이다.
```java
public class TrieRouter {
    private final TrieNode root = new TrieNode();

    public void insert(String pattern, Controller controller) {
        String[] parts = pattern.split("/");
        TrieNode node = root;

        for (String part : parts) {
            if (part.isEmpty()) continue;
            node = node.children.computeIfAbsent(part, k -> new TrieNode());
        }

        node.isPatternEnd = true;
        node.value = controller;
    }
}

```

### Trie 탐색
요청 URL을 / 기준으로 분리한 후, 루트부터 자식 노드를 순차적으로 따라가면서 탐색한다. 탐색 중 isPatternEnd가 true인 노드를 만날 때마다 가장 최근 매칭된 노드를 갱신한다. 탐색이 끝나면 가장 긴 접두사에 대응하는 매핑 객체를 반환할 수 있다.
탐색도 마찬가지로 O(L)시간에 탐색 가능하다.
```java
[유저 URL 요청]
Controller ctrl = router.match("/user/login/view/detail"); 
// "/user/login/view/*" 와 매칭됨 → LoginViewController

```

```java
public Controller match(String uri) {
    String[] parts = uri.split("/");
    TrieNode node = root;
    Controller lastMatched = null;

    for (String part : parts) {
        if (part.isEmpty()) continue;
        if (!node.children.containsKey(part)) break;

        node = node.children.get(part);
        if (node.isPatternEnd) {
            lastMatched = node.value;
        }
    }

    return lastMatched;
}

```

