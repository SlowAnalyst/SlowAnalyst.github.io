---
layout: post
title: "Go - TUI 라이브러리 BubbleTea"
summary: "Go 간단 TUI 라이브러리"
author: eveheeero
date: '2024-02-03 13:17:22 +0900'
category: ['go', 'learning']
tags: go
thumbnail: /assets/img/posts/default.png
keywords: go, tui, bubbletea
usemathjax: false
permalink: /blog/Go-BubbleTea/
---

## 개요

Go에는 [라이브러리 사이트](pkg.go.dev)를 통해 여러 라이브러리를 검색할 수 있다.

하지만 검색 사이트의 성능이 좋지 않아, 대부분 어떤 상황에 어떤 라이브러리가 좋은지 찾기 어려우며, 라이브러리를 찾아도 사용법을 모를 떄가 많다.

오늘은 BubbleTea Go TUI 라이브러리를 소개하고, 이에 대한 사용법을 소개한다.

## 소개

BubbleTea는 [charm.sh](charm.sh)에서 만든 TUI 라이브러리로, [charm.sh](charm.sh)의 여러 라이브러리 중 가장 주요하게 사용되는 라이브러리 입니다.

해당 라이브러리 외에도 [charm.sh](charm.sh)에서 텍스트 스타일링 라이브러리([LipGloss](https://github.com/charmbracelet/lipgloss)), 컴포넌트 라이브러리([Bubbles](https://github.com/charmbracelet/bubbles))등을 찾을 수 있으며, 간단한 입력 폼을 만들때는 폼 프레임워크([Hug](https://github.com/charmbracelet/huh))를 이용할 수 있습니다.

## 설치

BubbleTea는 사용하고자 하는 Go 프로젝트 폴더에서, 터미널로 `go get github.com/charmbracelet/bubbletea` 를 입력해 설치할 수 있으며,

보통 `import tea "github.com/charmbracelet/bubbletea"`를 통해 `tea`로 단축하여 사용합니다.

## 사용법

BubbleTea를 실행하는 방법은, `tea.Model`을 구현하고 있는 타입을 `tea.NewProgram(model).Run()`명령을 통해 사용할 수 있다.

```go
type model struct {
  // 데이터들이 이곳에 들어갑니다.
}
func (m model) Init() tea.Cmd {
  return nil // 초기화해주는 함수, 일반적으로는 nil을 사용
}
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
  // 입력이 일어났을 때, msg로 어떤 입력이 들어왔는지 들어온다
    // 로직을 처리한 후, 값을 수정하여 m 객체를 리턴한다.
  switch msg := msg.(type) {
  case tea.KeyMsg:
    switch msg.String() {
    case "ctrl+c":
    // 종료
    case "tab":
    // 다음
    case "shift+tab":
    // 뒤로가기
    }
  case tea.MouseMsg:
  // ...
  }
  return m, nil
  // return m, tea.Quit 로 프로그램을 종료시킬수도 있다. 하지만 그 외 다른 명령어들은 별로 없는 듯 하다.
}
func (m model) View() string {
  s := ""
  // 연산된 값을 가지고 출력할 내용을 반환한다.
  // 이곳에서 로그를 찍거나 출력을 하면 TUI가 깨지게 된다.
  return s
}
```

위와 같은 방식으로 TUI를 제작할 수 있으며, 이후 자유롭게 출력할 내용을 정하거나, 입력에 따라 연산을 진행할 수 있다.

> 기능은 많이 없지만 필요한건 있는 듯 하다.
>
> 추가적으로 필요한 내용이 있으면 직접 작서애향 할 것 같다.
>
> 무엇보다 버튼을 누르지 않으면 View가 업데이트 되지 않아서 별로였다.
>
> 누르지 않았을 때 View를 업데이트하려면 몇 이벤트를 던져야 할 듯 하다.
