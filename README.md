# URP_Portal
URP_Portal


### Portal 통해 서로를 볼 수 있는 재귀적 렌더링 시스템

<img width="995" height="558" alt="image" src="https://github.com/user-attachments/assets/bd3a71c4-2938-4e31-a5be-be7cbecefa1f" />


### 요약
1.	두 포탈이 서로의 뷰를 RenderTexture에 그림
2.	포탈 카메라를 반대편 포탈 위치로 변환
3.	Oblique Projection으로 포탈 표면에서 정확히 자름
4.	재귀적으로 7번 렌더링하여 무한 거울 효과 생성
5.	결과를 포탈 머티리얼에 표시

----
### 포탈이란

- 2개의 Portal 오브젝트가 서로 연결되어 있음
- 각 포털은 상대방 포털을 통해 보이는 장면을 RenderTexture에 렌더링
- portalCamera가 실제 렌더링을 수행하는 카메라


### 초기화
```C#
private RenderTexture tempTexture1;
private RenderTexture tempTexture2;

private Camera mainCamera;

private void Awake()
{
    mainCamera = GetComponent<Camera>();

    tempTexture1 = new RenderTexture(Screen.width, Screen.height, 24, RenderTextureFormat.ARGB32);
    tempTexture2 = new RenderTexture(Screen.width, Screen.height, 24, RenderTextureFormat.ARGB32);
}

private void Start()
{
    //각 포탈 메시의 머티리얼에 렌더 텍스처를 할당
    portals[0].Renderer.material.mainTexture = tempTexture1; 
    portals[1].Renderer.material.mainTexture = tempTexture2;
}

```

### 랜더링 이벤트에 연결
- 메인 카메라가 렌더링되기 전에 포탈 뷰를 먼저 렌더링

```C#
private void OnEnable()
{
    RenderPipeline.beginCameraRendering += UpdateCamera;
}

private void OnDisable()
{
    RenderPipeline.beginCameraRendering -= UpdateCamera;
}

```

### 랜더링 로직
```C#

void UpdateCamera(ScriptableRenderContext SRC, Camera camera)
{
    if (!portals[0].IsPlaced || !portals[1].IsPlaced) //포탈이 설치되지 않았으면 렌더링 안 함
    {
        return;
    }

    if (portals[0].Renderer.isVisible)
    {
        portalCamera.targetTexture = tempTexture1; //포탈 카메라의 출력을 tempTexture1로 설정
        for (int i = iterations - 1; i >= 0; --i)  // 재귀적으로 렌더링 (무한 반사 효과) - 역순 랜더링
        {
            RenderCamera(portals[0], portals[1], i, SRC);
        }
    }

    if(portals[1].Renderer.isVisible)
    {
        portalCamera.targetTexture = tempTexture2; 
        for (int i = iterations - 1; i >= 0; --i) 
        {
            RenderCamera(portals[1], portals[0], i, SRC);
        }
    }
}


```

### 카메라 변환
- 초기 카메라 위치 설정
- 포탈 변환 루프
- 경사투영
- 랜더링 실행

1. 초기 카메라 위치 설정
```C#
cameraTransform.position = transform.position;
cameraTransform.rotation = transform.rotation;
```

2. 포탈 변환 루프
- 플레이어가 포탈 A를 바라보고 있을 때,
- 플레이어의 위치를 포탈 A의 로컬 공간으로 변환
- 180도 회전(포탈을 통과한다고 가정)
- 포탈 B의 취리 기준으로 다시 월드 공간으로 변환
```C#
for(int i = 0; i <= iterationID; ++i)
{
    // 로컬 좌표로 변환
    Vector3 relativePos = inTransform.InverseTransformPoint(cameraTransform.position);
    // 180도 회전 (포탈 반대편)
    relativePos = Quaternion.Euler(0.0f, 180.0f, 0.0f) * relativePos;
    // 출구 포탈의 월드 좌표로 변환
    cameraTransform.position = outTransform.TransformPoint(relativePos);

    Quaternion relativeRot = Quaternion.Inverse(inTransform.rotation) * cameraTransform.rotation;
    relativeRot = Quaternion.Euler(0.0f, 180.0f, 0.0f) * relativeRot;
    cameraTransform.rotation = outTransform.rotation * relativeRot;
}
```

3. 경사 투영(Oblique Projection)
- 출구 포탈 표면에 클리핑 평면 생성
- 포탈 뒤쪽의 물체는 렌더링하지 않음.
```C#
Plane p = new Plane(-outTransform.forward, outTransform.position);
Vector4 clipPlaneWorldSpace = new Vector4(p.normal.x, p.normal.y, p.normal.z, p.distance);
```
- 클리핑 평면을 카메라 공간으로 변환
- 카메라의 투영 행렬를 왜곡하여 포탈 표면이 nearPlane이 되도록 조정
```C#
Vector4 clipPlaneCameraSpace =
    Matrix4x4.Transpose(Matrix4x4.Inverse(portalCamera.worldToCameraMatrix)) * clipPlaneWorldSpace;
```

4. 렌더링 실행
- urp에서 포탈 카메라를 한 번 렌더링
- 결과가 targetTexture에 저장
```C#
UniversalRenderPipeline.RenderSingleCamera(SRC, portalCamera);
```

### 재귀 렌더링 효과

iterations = 7일 때:
- 6번째 반복: 가장 깊은 포탈 안의 포탈
- 5번째 반복: 그 다음 깊이
- ...
- 0번째 반복: 첫 번째 포탈 뷰
- 깊은곳 부터 그려야 올바른 재귀 효과가 나옴.
