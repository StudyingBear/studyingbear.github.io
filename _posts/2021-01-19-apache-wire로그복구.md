---
title: apache-wire 로그 복구
date: 2021-01-19 22:10:00 +09:00
categories: [한 것, 기록용]
tags: [apache, wire, log]     # TAG names should always be lowercase
---

# Apache의 HttpClient(Wire)에서 출력한 로그 복구

Java로 multipart/form-data 를 보내야 할 일이 생겼는데, 실제로 HttpURLConnection을 통해 구현하자니 해야 할 일이 많아 Apache의 Httpclient를 사용하기로 하였다.

실제 개발때는 보내는 데이터가 맞게 들어갔는지 하나하나 확인은 못했는데, 개선과정에서 전송하는 데이터와 내용까지 알고싶다는 얘기가 나왔다.

Httpclient에서 Debug레벨로 지정해보니, 로그는 Apache의 Wire라는 클래스를 통해서 출력되고 있었다.

대괄호, 특수문자 등이 포함되어있길래 Wire클래스를 뜯어보니, 변환룰은 이하와 같았다.

```java
    class Wire {

    public static Wire HEADER_WIRE = new Wire(LogFactory.getLog("httpclient.wire.header"));
    
    public static Wire CONTENT_WIRE = new Wire(LogFactory.getLog("httpclient.wire.content"));
    
    /** Log for any wire messages. */
    private Log log;
    
    private Wire(Log log) {
        this.log = log;
    }
    
    private void wire(String header, InputStream instream)
      throws IOException {
        StringBuffer buffer = new StringBuffer();
        int ch;
        while ((ch = instream.read()) != -1) {
            if (ch == 13) {
                buffer.append("[\\r]");
            } else if (ch == 10) {
                    buffer.append("[\\n]\"");
                    buffer.insert(0, "\"");
                    buffer.insert(0, header);
                    log.debug(buffer.toString());
                    buffer.setLength(0);
            } else if ((ch < 32) || (ch > 127)) {
                buffer.append("[0x");
                buffer.append(Integer.toHexString(ch));
                buffer.append("]");
            } else {
                buffer.append((char) ch);
            }
        } 
        if (buffer.length() > 0) {
            buffer.append("\"");
            buffer.insert(0, "\"");
            buffer.insert(0, header);
            log.debug(buffer.toString());
        }
    }
    ...이하생략
```

정리한 패턴
* \r(캐리지리턴)의 경우는 문자열[\\r] 출력
* \n(라인피드)의 경우는 문자열[\\n] 출력 후 줄바꿈
* char 값이 32 미만이거나 127 초과 일 경우, `[0x` + Integer.toHexString + `]`
* 그 외의 경우는 char로 표현한 값을 로그로 출력

시도했던 방법
1. org.apache.http.Wire >> "ASDF[0xff]"와 같은 로그 -> apache.http.Wire 문자열이 존재할 경우, >> 이후의 문자열을 정규표현식으로 찾아 리스트에 보존
2. [\r], [\n] 문자열을 정규식으로 찾아 13, 10 으로 변환 후, 다시 처음부터 읽어서 변환 -> 바이트로 변환해야 할 대상의 길이가 어디까진지 알 수 없게 됨 -> 읽어들여서 변환 후에는 따로 취합해놔야 할 필요성을 느낌.
3. 문자열이 [라면, 뒤의 4문자까지 범위를 잡아 byte로 변환 -> [0xOO]같은 데이터만 예상했으나, [0xO]와 같은 한자리 데이터도 있어서 [로 시작하더라도 뒤의 문자열 확인의 필요성을 느낌.
4. 변환 결과가 뭔가 이상해서 찾아보니, [0x 가 아닌 패턴이 있었음. char 91 = [ 라서 2.의 방법에 [의 경우도 뒤의 문자열도 전부 확인해서 변환해야 한다는 결과가 나옴.

그래서 일단 대응한 방법
1. ```>>``` 이후 문자열을 잘라내서 리스트에 추가, 리스트에 추가한 문자열을 하나의 문자열로 변환
2. 리스트를 순회하면서, 리스트 내의 문자열을 이하의 룰에 맞춰서 변환
    - char 값이 [ 인지 아닌지 확인. 아니라면 byte로 변환 후, 리스트에 추가
    - char 값이 [ 일 경우 뒤의 문자 확인(패턴1) \r 일 경우 (byte) 13, \n 일 경우 (byte) 10으로 변환 후 리스트에 추가
    - char 값이 [ 일 경우 뒤의 문자 확인(패턴2) 0xO 혹은 0xOO 일 경우, 각각 값을 읽어들여서 Integer.parseUnsignedInt(값, 16)을 이용해서 byte로 변환 후 리스트에 추가
    - 이외의 경우는 한글자씩 읽어들여 byte로 변환 후 리스트에 추가
3. byte로 변환 된 리스트를 데이터 패턴에 맞춰 Image일 경우 Base64로 인코딩, 외에는 문자열 인코딩 적용 해서 각각 맵에 저장.

이하 대충 만들었던 소스. (이후 수정예정)

다만 문자열로 표현된 것을 임의로 룰을 정해서 복원(?) 한 것이라, 문제는 생길 것이라 생각되는데 별 수 있나.. [0xOO] 라는 값이 실제로도 6문자 일지도 모르는일이고