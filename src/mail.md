### 메일 인코딩&디코딩
- 메일파일에서 제목과 본문에 아래와 같은 형식을 볼 수 있음.
- 메일 제목과 본문은 UTF-8, BASE64로 인코딩 해야 함.
```
=?utf-8?B?7JWE7J207Y+wOC5qcGc=?=
    ^   ^
    |   +--- The bytes are Base64 encoded
    |
    +---- The string is UTF-8 encoded
```
- decode sample code
```java
    String encode = "=?utf-8?B?7JWE7J207Y+wOC5qcGc=?=";
    encode = encode.substring(encode.indexOf("B") + 2);
    encode = encode.substring(0, encode.indexOf("?"));
    encode = new String(encode.getDecoder().decode(encode.getBytes()));
    System.out.println(encode);
    //=> 아이폰8.jpg
```

### 메일 전송
- MimeMessageHelper을 이용한 메일 보내기
- MimeMessageHelper? MIME 메세지 지원과 Java에서 쉽게 메일을 보낼 수 있음.

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
```
```java
    @Autowired
	private JavaMailSender mailSender;

    public void sendMail(){
        MimeMessage message = mailSender.createMimeMessage();
	    message.setHeader("Content-Type", "text/html; charset=UTF-8");
	    message.addHeader("Content-Transfer-Encoding","base64");
        message.addHeader("Importance", "Normal");

        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");//multipart로 보내려면 true
        helper.setSubject("[공지] 마이링크 전체회식");
        /*
        받는사람 혹은 보낸사람에 닉네임을 넣어서 보내고 싶으면 
        InternetAddress 클래스를 이용하자.
        */
        InternetAddress to = new InternetAddress(); //InternetAddress에 닉네임과 그룹메일을 set할 수 있다.
        to.setAddress("jose@my-link.co.kr");
        to.setPersonal("조제(개발4팀)", "UTF-8");
		helper.setTo(to); //받는 사람
        /** 참조, 숨은참조 코드 생략.. (위와 동일)*/
        helper.setCc(cc);
        helper.setBcc(bcc);
        InternetAddress from = new InternetAddress();
        from.setAddress("naverpayadmin_noreply@navercorp.com");
        from.setPersonal("네이버페이");
		helper.setFrom(from); //보낸 사람
    
		helper.setPriority(3); //1은 중요메일, 3은 노멀, 5는 중요하지 않은 메일

        //본문 내용은 Multipart로 보내야 한다.
        Multipart multipart = new MimeMultipart();
        //본문 내용 추가
        BodyPart bodyPart = new MimeBodyPart(); 
        String contents = "<h1>안녕하세요. 네이버㈜, 네이버파이낸셜㈜ 입니다.</h1>";
        bodyPart.setContent(contents,"text/html;charset=UTF-8");
        multipart.addBodyPart(bodyPart);
        //첨부 파일 추가
        MimeBodyPart mimeBodyPart = new MimeBodyPart();
        File file = new File("/var/data/급여명세서.pdf");
        mimeBodyPart.attachFile(file);
        multipart.addBodyPart(mimeBodyPart);
        message.setContent(multipart); //본문내용 + 첨부파일 추가 (MimeMessageHelper에는 본문추가 메소드가 없음)
        mailSender.send(message); //메일 전송
    }
```
### IMAP
- 인터넷 메시지 액세스 프로토콜(IMAP)은 이메일을 받기 위한 프로토콜 중 하나.
```java
    public void connect(){
        Folder inbox = null;
		Store store = null;
        Session session = Session.getInstance(getMailProperties());
        store = session.getStore("imap");
        store.connect("userId", "password");
        inbox = store.getFolder("INBOX");
        System.out.println("새로운 메일 수 : "+inbox.getNewMessageCount());
        System.out.println("전체 메일 수 : "+inbox.getMessageCount());

        if (inbox != null) { inbox.close(false);}
		if (store != null) { store.close();}
    }
    private Properties getMailProperties() {
		Properties properties = System.getProperties();
		properties.setProperty("mail.store.protocol", "imaps");
		properties.setProperty("mail.host", MAIL_IP);
		properties.setProperty("mail.port", "25");
		properties.setProperty("mail.username", USER_NAME);
		properties.setProperty("mail.password", USER_PASSWORD);
		return properties;
	}
```

### 메일 받기
- MimeMessageParser를 사용하면 여러 도메인 메일을 파싱할 수 있다.

```java
    public void parseMail(){
        Folder inbox = null;
		Store store = null;
		Session session = Session.getInstance(getMailProperties());
		store = session.getStore("imap");
		store.connect(userId+domain, password);
		inbox = store.getFolder("INBOX");
        Message[] messages = inbox.getMessages();//imap을 이용해 메시지를 가져옴.
        for(int i=0; i<messages.length; i++){
            MimeMessage mimeMessage = (MimeMessage) messages[i];
            MimeMessageParser parser = new MimeMessageParser(mimeMessage);
            String imporatance = mimeMessage.getHeader("Importance")[0].toUpperCase();//HIGH, NORMAL, LOW
            mimeMessage.getMessageID(); //메일 ID
            parser.getSubject(); //메일제목
            //보낸 사람
            Address[] address = mimeMessage.getFrom();
            InternetAddress from = (InternetAddress) address[0];
            from.getAddress(); //주소
            from.getPersonal(); //닉네임
            //받는 사람
            parser.getTo(); //parser를 이용해서 받을 수 있으나 닉네임과 주소를 따로 받을 수 없다.
            //참조 
            parser.getCc();

            //본문 내용 + 첨부파일 파싱
            Object o = messages[i].getContent();
            if (o instanceof Multipart) {
                Multipart multi = ((Multipart) o); //Multipart multipart/mixed
                for (int j = 0; j < multi.getCount(); j++) { //multi.getCount() : 본문내용과 첨부파일 수
                    MimeBodyPart part = (MimeBodyPart) multi.getBodyPart(j);
                    if(part.isMimeType("text/html")) { //내용만 있는 메일
                        part.getContent().toString();
				    }  
                    if(part.isMimeType("multipart/alternative")){ //내용과 첨부파일이 포함된 메일
                        //해당 메일은 Multipart를 multipart/alternative 바꾼 상태로 파싱을 해야한다.
                        Multipart mpAlternative = (Multipart) b.getContent(); //Multipart multipart/alternative
                        MimeBodyPart part = mpAlternative.getBodyPart(1);//text/html은 1번 째에 있음.
                        if(part.isMimeType("text/html")) {
                            part.getContent().toString(); //내용 파싱
				        }
                    }
                    if (part.getDataHandler().getName() != null) { //첨부파일
                        String fileName = part.getDataHandler().getName();
                        if (fileName.contains("=")) { //첨부파일명 디코드
                            fileName = fileName.substring(fileName.indexOf("B") + 2);
                            fileName = fileName.replaceAll("[=?]", "");
                            Decoder decode = Base64.getDecoder();
                            byte[] decodeByte = decode.decode(fileName.getBytes());
                            fileName = new String(decodeByte);
					    }
                        File file = new File("/var/data/" + fileName);
                        OutputStream out = new FileOutputStream(file);
                        InputStream in = part.getInputStream();
                        int k;
					    while ((k = in.read()) != -1) {
						    out.write(k);//파일 저장
					    }
                    }
                }
            }
        }
        if (inbox != null) { inbox.close(false);}
		if (store != null) { store.close();}
        if (in != null) { in.close();}
		if (out != null) { out.flush(); out.close();}
    }
```