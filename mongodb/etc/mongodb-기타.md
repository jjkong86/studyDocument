mongodb 4.0
 - 4.0 이전 version 에선 non-blocking secondary read 기능이 없음.
 - write가 primary에 반영되고 secondary들에 다 전달 될때까지 secondary는 read block해서
      데이터가 잘못된 순서로 read되는 것을 방지하고 있음.
 - 그래서 write가 순간적으로 몰리면 read성능 저하를 유발함.
 - 4.0 부터는 data timestamp와 consistent snapshot을 이용하여 이슈를 해결함.


