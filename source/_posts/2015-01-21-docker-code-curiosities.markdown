---
layout: post
title: "Docker code curiosities"
date: 2015-01-21 08:17:51 +0100
comments: true
categories: [ docker, devops ]
---

Reading the code is the best documentation place at the end, and despites the fact that it is time consuming and sometimes hard to understand, it could make you happier sometimes.

This post is about the unrelated things I found (for now) in docker code.

### The meaning of life

Given the name of the blog, this one did not scape to my eyes...

``` go meaning_of_life https://github.com/docker/docker/blob/master/engine/streams_test.go
func TestOutputAddEnv(t *testing.T) {
	input := "{\"foo\": \"bar\", \"answer_to_life_the_universe_and_everything\": 42}"
	o := NewOutput()
	result, err := o.AddEnv()
	if err != nil {
		t.Fatal(err)
	}
	o.Write([]byte(input))
	o.Close()
	if v := result.Get("foo"); v != "bar" {
		t.Errorf("Expected %v, got %v", "bar", v)
	}
	if v := result.GetInt("answer_to_life_the_universe_and_everything"); v != 42 {
		t.Errorf("Expected %v, got %v", 42, v)
	}
	if v := result.Get("this-value-doesnt-exist"); v != "" {
		t.Errorf("Expected %v, got %v", "", v)
	}
}

```

Remember, don't forget your towel!


### Names generator

I enjoyed this wink, I mean, the whole file. It is worth to read it all, since all the names are quoted and this quick reference could wake up some interest about someone.

``` go things_clear https://github.com/docker/docker/blob/master/pkg/namesgenerator/names-generator.go 
begin:
	name := fmt.Sprintf("%s_%s", left[rand.Intn(len(left))], right[rand.Intn(len(right))])
	if name == "boring_wozniak" /* Steve Wozniak is not boring */ {
		goto begin
	}
```

I have to say I agree.




At least they are not as hard as the [ones](http://www.vidarholen.net/contents/wordcount/) in linux code. 
