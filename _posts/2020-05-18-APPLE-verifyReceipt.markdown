---
layout: post
title:  "애플로그인 인앱 결제 영수증 검증 , JAVA"
description: 앱에서 결제한 영수증 정보 백엔드 처리  
date:   2020-05-18 14:40:00 +000
categories: Java 
---
# 인앱 결제 
앱을 개발 및 서비스 하시는 분들이라면 잘아시겠지만 디지털 상품의 경우 애플에서는 애플스토어의 결제 시스템이 아닌 다른 결제 시스템을 허략하지 않습니다. (나ㅃ.. 으음..) 
그래서 간략하게 백엔드에서 영수증 처리를 하는 부분을 다뤄 볼까 합니다. 

우선 사용한 의존성은 다음과 같습니다. 
```xml
<dependency>
    <groupId>com.mashape.unirest</groupId>
    <artifactId>unirest-java</artifactId>
    <version>1.4.9</version>
</dependency>
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.0</version>
</dependency>
```

애플에서 제공하는 영수증 API 목록 <br>
https://buy.itunes.apple.com<br>
https://sandbox.itunes.apple.com<br> 

애플 API 가이드 <br>
https://developer.apple.com/documentation/appstorereceipts/verifyreceipt <br>


buy는 라이브 서비스용도 이며 sandbox 는 테스트 용도 입니다. <br>
ios 셋팅에서 샌드박스 아이디를 등록하면 테스트 진행시 편하게 테스트 진행을 할수 있습니다. <br>

먼저 간략하게 영수증을 검증하는 순서는 다음과 같습니다. <br>
영수증 검증요청 -> buy.itunes.apple.com 을 통하여 영수증 검증 요청 <br>
-> 리턴 값이 21007인경우 sandbox.itunes.apple.com 으로 영수증 검증 재요청 <br>
-> 검증 처리 및 완료 <br>

영수증을 Decode 하는 역활로 수행이 되는 부붑이기 때문에 인앱 결제상 이미 결제가 되어 있는 상태가 됩니다. <br>

일반적으로 사용하는 부분에 대해서 변수지정을 다음과 같이 하였습니다. <br>
```java
String itunesUrl = "https://buy.itunes.apple.com";
String sandboxUrl = "https://sandbox.itunes.apple.com" ;
/**
*애플에서 받은 키 정보 
*/
String appleSecretKey = "";
/**
*영수증 정보 
*/
String receiptData = "";
```

그리고 영수증을 검증하는 부부은 다음과 같습니다. 
```java
/* 영수증 정보 및 애플 키 변수 생성  */
JSONObject bodyData = new JSONObject()
    .put("receipt-data", receiptData)
    .put("password", appleSecretKey)
    .put("exclude-old-transactions", true);

/* 검증 요청  */
HttpResponse<JsonNode> response = Unirest.post(itunesUrl+"/verifyReceipt")
    .header("Content-Type", "application/json")
    .body(bodyData)
    .asJson();

/* sandbox 영수증인 경우  sandbox로 결제 검증 재요청 */
if(response.getStatus() == 200 && response.getBody().getObject().get("status").toString().equals("21007")) {
    response = Unirest.post(sandboxUrl+"/verifyReceipt")
            .header("Content-Type", "application/json")
            .body(bodyData)
            .asJson();
}
/* 영수증 검증이 정상처리된 경우  */
if(response.getStatus() == 200 && response.getBody().getObject().get("status").toString().equals("0")) {
    JsonParser parser = new JsonParser();
        JsonObject object = (JsonObject)parser.parse(response.getBody().getObject().get("receipt").toString());
        JsonArray array = (JsonArray)parser.parse(object.get("in_app").toString());
        // 최근 결제 내역 가져오기 
        String transaction_id = null;
        String original_transaction_id = null;           
        String purchase_date_ms = null;            
        String product_id = null;
        for(int i = 0 ; i< array.size() ; i ++) {
            object = (JsonObject)parser.parse(array.get(i).toString());
            if(transaction_id == null  || Long.parseLong(purchase_date_ms) < Long.parseLong(object.get("purchase_date_ms").toString().replaceAll("\"",""))) {
                transaction_id  = object.get("transaction_id").toString().replaceAll("\"","");
                original_transaction_id = object.get("original_transaction_id").toString().replaceAll("\"","");
                purchase_date_ms = object.get("purchase_date_ms").toString().replaceAll("\"","");  
                product_id = object.get("product_id").toString().replaceAll("\"","");
            }
        }
        System.out.println(transaction_id);
        System.out.println(original_transaction_id);
        System.out.println(purchase_date_ms);
        System.out.println(product_id);
}
```
코드를 보시면 결제 검증 이후 for 문이 의아해 하실수 있습니다. <br>
애플의 결제 영수증은 결제한 건에 대한 영수증 내역이 아닙니다. 해당 서비스를 통해 결제를 할 내역이라고 보시면 될듯합니다. <br>
하여 저같은 경우 for 문을 통하여 최근 처리된 영수증 내역을 가져오도록 하겠습니다. <br>
부가적으로 더 처리를 추가 한다면 취소된 내역은 제외 한다던지 하는 요소가 필요 합니다. <br>  

위의 코드는 영수증 검증시 필요한 내용에 대한 확인을 위한 코드 입니다.  <br>
<a href="https://github.com/ChoDaeYun/apple-verifyReceipt" target="_blank">https://github.com/ChoDaeYun/apple-verifyReceipt</a>

# 기타  
- 소모성 , 비소모성 타입의 상품에 대한 환불   <br>
    ( 소모성의 경우 환불 확인이 힘들다는 내용들이 많아 비소모성으로 상품을 등록하였으며 환불에 대한 체크를 IOS 클라이언트 레벨에서 영수증 <br>확인하여 환불 내역이 있을 경우 서버로 환불처리 요청을 하도록 개발을 하였습니다.  )<br>
- 정기 결제에 대한 구독 취소 , 환불 , 결제<br>
    (정기결제의 경우 구독취소는 상품은 유효하지만 재결제는 안한다는 내용이기에 이력 외의 별도의 서버에서 크게 처리 할 부분은 없습니다. <br>
     문제는 환불 , 재결제에 대한 이슈 인데 아직 실제로 개발은 진행해본적이 없어 고민만 하고 있습니다. )<br>
- 구글인앱결제와의 차이 .. <br>
    구글은 48시간 이후 서비스사에 문의 하도록 안내 되는데.. 애플 환불은 애플스토어에서만 가능합니다. <br>
    서비스사에서 환불을 처리 해주는게 가능한지도 모르겠습니다. 관련 내용도 찾지 못해서 .. <br>
    구글은 재무관리 권한이 있다면 주문내역조회 가 가능하여 환불진행이 필요 할때 서비스사에서 직접 환불이 가능한데 말입니다. <br>
    그 외에 일반적인 결제 서비스를 봐도 별도의 환불 기능이 존재 하는데 애플은 왜 없는거 같은지.. <br>

