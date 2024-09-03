- counterfactual 개념  
    - section 9.3 : https://christophm.github.io/interpretable-ml-book/counterfactual.html
    - 한글 버전 : https://eair.tistory.com/32
- 모바일 데이터 기반의 causal inference"의 경우, 아래의 튜토리얼 논문을 참고하기  
https://doi.org/10.1145/3648356  
    ~~~
    예시에서는 스마트폰 앱 사용과 신체활동 사이의 인과관계 추론에 대해서 설명하였는데, 비슷한 원리로 생활 습관(e.g., 신체활동, 식습관, 수면 등등)과 glucose level 사이의 인과관계 추론으로도 이어질 수 있음.
    ~~~
- 또한, intervention까지는 아니지만, 이러한 "인과관계 결과를 어떻게 사용자에게 제공하면 좋을지"에 대한 연구  
https://doi.org/10.1145/3613904.3642766  
    ~~~
    위의 튜토리얼 논문에서 사용했던 방법론을 그대로 적용했으며, 여기서는 사람들이 어떻게 인과관계를 이해-해석-활용하는지에 대해, 사용자 경험을 주로 살펴보았습니다
    intervention까지 넘어간다면, "그래서 (나는) 어떻게 해야하는지", "왜 그렇게 해야하는지"에 대한 설명을 잘 제공하는 것도 중요할 것으로 보입니다
    ~~~
- intervention 관련 연구  
https://doi.org/10.1145/3328910  
    ~~~
    Just-in-time adaptive intervention (JITAI)이라는 개념이 들어가있는데, 사용자가 딱 필요한 순간에 intervention을 제공하는 것에 대한 연구입니다
    intervention을 주더라도, 그게 실제로 사용자의 행동 변화에 이어지고, 실질적인 건강 상태의 증진에 이어지기까지는 고려해야 할 허들이 꽤 있습니다 -- 이런 것들에 대해 한 번 생각해볼 수 있는 연구입니다
    ~~~
