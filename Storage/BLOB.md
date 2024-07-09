## BLOB Storage
BLOB(Binary Large Object)는 비정형화된 데이터를 주로 저장하는 데 사용하는 스토리지이다. 다른 Storage(Database, File System)에 비해서 빠르고 확장이 용이하다는 장점이 있다.

> 아래 글에서 설명하는 BLOB Storage와 Object Storage라는 단어는 같은 뜻이다.
---
&nbsp;

### BLOB Storage(Object Storage)란 뭘까?
백엔드 개발을 하다보면 주로 접하게 되는 Database나 File System과 같이 데이터를 저장하기 위한 수단이다. 각 수단은 용도가 있는데, Object Storage는 주로 대량의 비정형 데이터를 처리하기 위한 Storage이다.

&nbsp;
> 그렇다면 Object 라는건 뭘까? => 그저 데이터 덩어리 일뿐이다. 이미지가 될 수도 있고, 비디오파일, 한글 파일, 혹은 소프트웨어 전체, 가상 머신 등.. 말그대로 이진데이터 덩어리이다.

아래는 클라우드에서 제공되는 대표적인 BLOB Storage 서비스 이다.
- Amazon S3
- Google Cloud Storage
- Azure Blob Storage

### Storage 비교
Object 스토리지는 비정형화된 데이터를 다루고 빠르고 확장 가능하다는 장점이 있다고 했다. 그렇다면 다른 Storage들에 Object 데이터를 저장한다고 가정하고 비교를 해보자.

1. DB(DataBase)
100MB짜리 Video를 Binary 형태로 DB에 저장한다고 생각해보자. 게다가 그러한 요청이 100개, 1000개로 점점 늘어난다면 DB Performance issue가 생길 수 밖에 없다. DB는 애초에 수 많은 데이터 Access 요청을 빠르게 처리하는 용도로 설계 되었기 때문에 대용량의 Binary 파일들을 저장하는데는 적절하지 않다. (속도문제)

2. FileSystem
FileSystem은 비정형 데이터들을 저장하기에 나쁘지 않다. 과거에는 파일들을 위한 FTP서버를 두고 서버내부에 FileSystem에 데이터를 저장하고 보내고 하며 사용하기도 했다. 지금도 이것은 나쁜방법이 아니지만 Blob Storage와 비교한다면 확장성이 떨어진다.

FileSystem은 계층형 구조로 되어있다. 말하자면 트리구조로 되어있다는 것이다. 이에 반해 Blob Storage는 flat하고 비정형적인 구조를 띈다. 그렇기 때문에 확장하기가 훨씬용이하다. 
> 상상이 잘안간다면 컴퓨터 용량이 부족해서 SSD를 하나더 사서 부착한 경우를 생각해보자. C드라이브, D드라이브로 나눠사용하거나 파티션 설정 같은 작업들을 해줘야 하는데 Blob Storage는 그럴 필요가 없다. 애초에 구조랄게 없기 때문이다.

### Blob Storage의 특징
1. Flat
말 그대로 평평하다. 위 문단에서 File System은 트리형태의 계층구조라고 했다. Blob은 그 반대다. 그런데 Blob서비스들을 사용하다 보면 Folder같은 느낌을 제공하곤한다. 사실 이는 눈속임이다. 그저 폴더처럼 보일 뿐이지 실제로는 그렇지 않다. 코드에서도 이를 조금 엿볼수 있다. 아래는 S3의 데이터를 불러오는 typescript 코드 스니펫이다.

```typescript
  async function getFileNames(path: string) {
    const setBucketAndPath = {
      Bucket: this.BUCKET_NAME,
      Prefix: `${path}`,
    };
    const getS3Objects: Promise<AWS.S3.ObjectList> = new Promise(
      (resolve, reject) => {
        this.s3Client.listObjects(setBucketAndPath, (err, result) => {
          if (err) return reject(err);
          resolve(result.Contents);
        });
      },
    );
  }
```
setBucketAndPath의 Property를 보면 Prefix라고 되어있다. 이처럼 같은 prefix로 folder인 것 마냥 눈속임을 하는것이지 실제로는 그렇지 않다.

2. Identifier
Blob Storage의 데이터들은 모두 다른 identifier를 가져야한다. 이는 Blob이 Flat하기 때문에 각각의 Object를 구분하기 위해서 당연한 것이다. 
 
### 실제 사용
필자는 Blob Storage를 사용할때 Blob에 접근가능한 요소들 (fileName이나 URI등)을 저장해두고 SDK를 사용해서 Stream Pipe를 연결해주거나, Client에서 직접 Blob으로 요청할 수 있도록 Signed URL을 주는 방식으로 사용한다. 
