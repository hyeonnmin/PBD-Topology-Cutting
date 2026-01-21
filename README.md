# 📚 Position Based Dynamics Cloth Balloons with Boundary Particles

## Overview

**Position-Based Dynamics(PBD)** 를 기반으로 한 실시간 Cloth Balloon 시뮬레이션에서,
Boundary Particle 구조를 활용하여 **topology change**가 가능한 cutting 기능을 구현한다.

2D 입력 벡터를 3D cutting plane으로 변환하고, 이를 기준으로 boundary particle의 연결을 분리함으로써 tearing을 처리한다.

---

## 📌 Problem Statement & Motivation

PBD 기반 시뮬레이션은 topology가 고정되어 있다는 가정하에 설계되는 경우가 많아, 찢어짐이나 절단과 같은 구조적 변화에 취약하다.
본 프로젝트는 이러한 한계를 해결하기 위해 topology change를 명시적으로 다루는 cutting 기능을 구현한다.

---

## 📌 Core Idea

본 프로젝트의 핵심 아이디어는 topology change를 직접적인 mesh 조작이 아닌, 기하학적으로 정의된 cutting plane 문제로 변환하는 것이다.

이를 통해 PBD 시뮬레이션에서 암묵적으로 고정되어 있던 topology를 명시적으로 분리하고 제어할 수 있도록 한다.

사용자 또는 외부 상호작용으로부터 입력되는 cutting 정보는 2D 벡터 형태로 주어지며, 이는 월드 좌표계에서의 **3D cutting plane**으로 확장된다.

해당 plane은 시뮬레이션 전반에서 일관된 분리 기준으로 사용되며, boundary particle로 구성된 Cloth Balloon 표면을 공간적으로 양분한다.

---

## 📌 System Architecture

본 시스템은 사용자 입력(마우스)으로부터 **cutting 요청을 생성**하고, boundary particle 기반의 Cloth Balloon topology를 **검출–분리–재초기화**하는 파이프라인으로 구성된다. 전체 흐름은 다음과 같다.


<img width="629" height="437" alt="제목 없는 다이어그램 drawio (1)" src="https://github.com/user-attachments/assets/3156e8f5-9b9b-451b-a5bc-781682bf0fdb" />


## Runtime Pipeline

### 1. Input Event Layer
  - **Mouse Button Down** 이벤트 발생 시, 화면 입력을 기반으로 cutting 방향/구간을 결정한다.
  - 입력 정보는 `LineCut()`으로 전달된다.

### 2. Cutting Orchestration
 - `LineCut()`은 cutting 동작의 상위 오케스트레이터 역할을 하며, 아래 단계들을 순차적으로 수행한다.
 - cutting plane(또는 line cut에 대응되는 분리 기준)을 구성하고, 해당 기준을 모든 geometry/topology 연산에서 공통으로 사용한다.

### 3. Intersection Collection
  - `CheckCut()`:
    - cutting plane과 교차하는 **edge / vertex 후보**를 탐색하고 기록한다.
    - 이후 단계의 분리 연산이 일관되게 수행될 수 있도록, 교차 정보(교차 위치, 대상 edge/vertex, plane side 등)를 축적한다.

### 4. Topology Split
  - `CutEdge()`:
    - 교차한 edge에 대해 새로운 **vertex**를 생성하고,
    - plane의 양쪽 영역이 섞이지 않도록 **edge 연결을 분리**한다.
  - `CutVertex()`:
    - vertex 기반으로 교차/절단이 정의되는 경우, **vertex를 복제하여 양쪽 topology로 분리**시킨다.
    - 삼각형/인접 정보가 일관되게 유지되도록 업데이트한다.

### 5. Reinitialization

  - Cutting 이후 topology 구조가 변경되므로, 시뮬레이션 및 렌더링에 필요한 데이터를 재구성한다.

  - `Reinitialization()`:
    - `InitIndex()` : 갱신된 vertex/triangle 구성에 맞춰 index buffer(또는 triangle list)를 재생성
    - `InitEdge()` : edge adjacency를 재구축(이후 constraint/충돌/후처리에 사용)
    - `UpdateNormal()` : 새 topology에 맞는 normal 재계산
    - `InitParticle()` : boundary particle 및 관련 constraint/속성 데이터를 재초기화(혹은 갱신)

---

## 📌 Key Implementation Details

본 섹션에서는 topology cutting 파이프라인을 구성하는 핵심 함수인
CheckCut(), CutEdge(), CutVertex()의 역할과 동작 방식을 간략히 설명한다.
Cutting은 **충돌 검출 → topology 분리**의 두 단계로 분리되며, edge 기반 분리와 vertex 기반 분리를 명확히 구분하여 처리한다.

---

### 1. `CheckCut()` – Cutting Intersection Detectio

`CheckCut()` 함수는 cutting plane과 boundary topology 간의 **교차 여부를 사전에 검출하고 기록**하는 역할을 한다.
  - 모든 boundary edge를 순회하며 cutting plane과의 교차 여부를 판단
  - 교차가 발생한 경우:
    -  교차 위치
    -  대상 edge 또는 vertex
    -  cutting plane의 양쪽 영역 정보 저장
  - edge 내부 교차와 vertex 직접 교차를 구분하여 이후 단계에서 서로 다른 분리 로직이 적용될 수 있도록 한다.
이 단계에서는 topology를 수정하지 않으며,
CutEdge / CutVertex 단계에서 일관된 분리가 이루어질 수 있도록 **충돌 정보를 기록**한다.

---

### 2. `CutEdge()` – Edge-based Topology Splitting

`CutEdge()`는 `CheckCut()`에서 기록된 edge 내부 교차 케이스를 처리한다.
  - cutting plane이 edge 내부를 통과하는 경우:
    - 교차 위치에 새로운 vertex(또는 particle)를 생성
    - 기존 edge를 두 개의 edge로 분할
    - cutting plane의 양쪽에 위치한 topology가 서로 연결되지 않도록 adjacency를 재구성

이 과정은 plane 기준으로 topology를 명확히 분리하는 것을 목표로 하며,
이후 constraint 재구성을 통해 분리된 구조가 물리적으로 독립적으로 동작하도록 한다.

---

### 3. `CutVertex()` – Vertex-based Topology Splitting

`CutVertex()`는 cutting plane이 기존 **vertex를 직접 통과하는 경우**를 처리한다.
  - plane 위 또는 plane과 매우 근접한 vertex를 기준으로:
    - 해당 vertex를 복제하여 두 개의 vertex로 분리
    - 각각의 vertex가 plane의 서로 다른 쪽 topology에 속하도록 재할당
  -  vertex를 공유하던 triangle 및 edge 연결 관계를 재구성하여
    **topology 간 직접적인 연결을 완전히 제거**

이 단계는 edge 기반 분리만으로 처리할 수 없는 case를 안정적으로 처리하기 위해 필요하다.

---

### 4. Cutting Cases

본 구현에서는 cutting 상황을 다음과 같은 4가지 대표적인 경우로 분류하여 처리한다.

1. Edge Interior Cutting
   - cutting plane이 edge 내부를 통과하는 경우
    </details>

    <details>
      <summary><b>Example</b></summary>
    <img width="300"alt="image" src="https://github.com/user-attachments/assets/673b9965-04d3-407b-8736-80b3aae8b14f" />

    **Before**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/ee1daceb-afb9-43d8-8964-f2109f72944f" />

    **After**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/f267a5cf-ba17-4658-a5b9-0bb2e7916256" />
    
    </details

2. Vertex Cutting
   - cutting plane이 기존 vertex와 직접 교차하는 경우
    </details>

    <details>
      <summary><b>Example</b></summary>
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/030a508b-42a7-4981-899c-b5a90f1de640" />

    **Before**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/c4a219d9-2e2b-4a2a-b914-5142bb8ab685" />

    **After**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/253771a8-c0b1-4216-98cb-2080ea40c309" />

    
    </details
    
3. Multiple Edge Intersections
   - 하나의 triangle에서 둘 이상의 edge가 동시에 절단되는 경우
    </details>

    <details>
      <summary><b>Example</b></summary>
    <img width="300"  alt="image" src="https://github.com/user-attachments/assets/a345c1fc-7e47-42b4-b84c-5be45694a6db" />

    **Before**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/e95dcdba-208b-41fa-b1f3-61ca38585e1f" />


    **After**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/e746f142-839e-45c6-8c16-8e982cafb2a4" />

   
    </details
4. Mixed Edge–Vertex Cutting
   - edge 내부 교차와 vertex 교차가 동시에 발생하는 경우
        </details>

    <details>
      <summary><b>Example</b></summary>
    <img width="300"  alt="image" src="https://github.com/user-attachments/assets/8aeff9b5-e8b6-4889-b390-8783f340d1b9" />

    **Before**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/e95dcdba-208b-41fa-b1f3-61ca38585e1f" />

    **After**
    <img width="300" alt="image" src="https://github.com/user-attachments/assets/9051a013-053e-4500-8a48-ddc3096f31d6" />

   
    </details
---

## 📌 실행 예시 및 샘플 출력 

https://github.com/user-attachments/assets/7dcde31d-3793-4f33-9347-d53c2249054e

---

## 📌 Limitations & Trade-Offs



## 📌 Future Work


---

## Development Environment

- Language: C++
- Graphics API: DirectX11
- Development Environment: Visual Studio 2022
- Build Configuration: x64, Debug / Release

### Hardware
- CPU: Intel CPU (Desktop)
- GPU: NVIDIA RTX 3060
- RAM: 16GB

### Platform
- OS: Windows 10 / 11
  
---

## References

- Müller et al., *Position Based Dynamics*, VRIPHYS 2006, AGEIA
- Akinci et al., Coupling Elastic Solids with SPH Fluids, SCA 2012
