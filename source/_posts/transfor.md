---
title: golang 替换string中的emoji、英文符号等
date: 2023-08-31 18:56:46
tags:
  - Golang
categories:
  - 技术
---

闲话不多说，今天我们主要解决的问题就是在golang中**如何替换字符串中的emoji、英文符号、中文符号、数字**。主要的解决办法还是通过正则表达式哈！
```go
package main

import (
	"net/url"
	"regexp"
	"strings"
)

var ChinsesPuncts = []string{
	"，", "。", "、", "！", "？", "：", "；", "﹑", "•", "＂", "…", "‘", "’", "“", "”", "〝", "〞",
	"∕", "¦", "‖", "—", "　", "〈", "〉", "﹞", "﹝", "「", "」", "‹", "›", "〖", "〗", "】", "【", "»", "«", "』",
	"『", "〕", "〔", "》", "《", "﹐", "¸", "﹕", "︰", "﹔", "！", "¡", "？", "¿", "﹖", "﹌", "﹏", "﹋", "＇", "´",
	"ˊ", "ˋ", "―", "﹫", "︳", "︴", "¯", "＿", "￣", "﹢", "﹦", "﹤", "‐", "­", "˜", "﹟", "￥", "﹩", "﹠", "﹪",
	"﹡", "﹨", "﹍", "﹉", "﹎", "﹊", "ˇ", "︵", "︶", "︷", "︸", "︹", "︿", "﹀", "︺", "︽", "︾", "ˉ", "﹁", "﹂",
	"﹃", "﹄", "︻", "︼", "（", "）",
}

// ReplaceEmoji 去除句子中的emoji表情
func ReplaceEmoji(sentence string) string {
	sentence = strings.TrimSpace(sentence)
	if sentence == "" {
		return ""
	}
	return regexp.MustCompile(`[\p{So}\p{Sk}]`).ReplaceAllString(sentence, "")
}

// ReplacePunct 去除句子中的标点符号
func ReplacePunct(sentence string) string {
	sentence = strings.TrimSpace(sentence)
	if sentence == "" {
		return ""
	}
	// 去掉英文标点符号
	sentence = regexp.MustCompile(`[[:punct:]\s]`).ReplaceAllString(sentence, "")
	sentence = url.QueryEscape(sentence)
	sentence = regexp.MustCompile(getChinsesPunctExp()).ReplaceAllString(sentence, "")
	sentence, _ = url.QueryUnescape(sentence)
	return sentence
}

// 返回中文标点符号的正则表达式
func getChinsesPunctExp() string {
	var chars []string
	for _, c := range ChinsesPuncts {
		chars = append(chars, url.QueryEscape(c))
	}
	str := strings.Join(chars, "|")
	return "(" + str + ")"
}

// 去除掉数字
func ReplaceNumeric(sentence string) string {
	sentence = strings.TrimSpace(sentence)
	if sentence == "" {
		return ""
	}
	return regexp.MustCompile(`[0-9]`).ReplaceAllString(sentence, "")
}

func main() {
	ReplacePunct("大家好ab!！！这是替换后的")
	ReplacePunct("hello!this is a test demo")
	ReplaceEmoji("😀 🆒 how are you? 🌞")
	ReplaceNumeric("这是1，2，3，4")
	
}
```
