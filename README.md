# Study-guide-3dsg

# 📚 Study Guide: 3D Scene Graphs for Robotics

> **Mục tiêu:** Tự học sâu về 3D Scene Graphs — từ nền tảng SLAM đến LLM-based planning.
> **Papers:** SayPlan, Kimera, Hydra, Open3DSG
> **Cách dùng:** Đọc outline → trả lời câu hỏi → code theo hướng dẫn → ghi chú lại kiến thức.

---

## 🗺️ Roadmap tổng quan

```
Week 1: Nền tảng (Kimera)
  └─> SLAM pipeline, VIO, Pose Graph, Mesh, Semantics

Week 2: Real-time Scene Graph (Hydra)
  └─> GVD, Places, Rooms, Loop Closure, Deformation Graph

Week 3: LLM + Scene Graph (SayPlan)
  └─> Grounding, Semantic Search, Iterative Replanning

Week 4: Open-Vocabulary (Open3DSG)
  └─> VLM Distillation, GNN, CLIP, LLM for relationships
```

---

## 📖 Module 1: Nền tảng SLAM & Metric-Semantic Mapping (Kimera)

### 1.1 Dàn ý kiến thức cần nắm

```
A. Visual-Inertial Odometry (VIO)
   ├── IMU preintegration là gì? Tại sao cần?
   ├── Feature detection (Shi-Tomasi) vs tracking (Lucas-Kanade)
   ├── Stereo matching & geometric verification (RANSAC)
   ├── Factor graph & fixed-lag smoothing
   └── iSAM2 solver trong GTSAM

B. Robust Pose Graph Optimization (RPGO)
   ├── Loop closure detection bằng Bag-of-Words (DBoW2)
   ├── Outlier rejection: PCM (Pairwise Consistent Measurement)
   ├── Chi-squared test cho odometry consistency
   └── Maximum clique để tìm tập loop closure nhất quán

C. 3D Mesh Reconstruction
   ├── TSDF (Truncated Signed Distance Function)
   ├── ESDF (Euclidean Signed Distance Function)
   ├── Marching Cubes algorithm
   ├── Per-frame mesh vs Multi-frame mesh
   └── Voxblox library

D. Semantic Annotation
   ├── Bundled raycasting
   ├── Bayesian label update per voxel
   ├── Dense stereo (Semi-Global Matching)
   └── 2D semantic segmentation → 3D propagation
```

### 1.2 Câu hỏi sâu

```
[Câu hỏi hiểu bản chất]
Q1. IMU preintegration giải quyết vấn đề gì so với integrate IMU
    measurement trực tiếp? Viết lại công thức preintegration factor.

Q2. Tại sao Kimera dùng "structureless vision model" — eliminate 3D
    points khỏi state? Lợi ích về complexity là gì?

Q3. Giải thích tại sao TSDF cần "truncation distance". Nếu không
    truncate thì chuyện gì xảy ra với noise?

Q4. Trong Bayesian semantic label update, prior và likelihood được
    định nghĩa như thế nào? Tại sao bundled raycasting chỉ update
    label "near the surface"?

[Câu hỏi so sánh]
Q5. So sánh fixed-lag smoothing vs full smoothing:
    - Memory & compute trade-off?
    - Khi nào nên dùng cái nào?

Q6. So sánh surfel-based (ElasticFusion) vs TSDF-based (Kimera)
    vs point-based representation. Ưu nhược điểm từng loại?

[Câu hỏi thiết kế]
Q7. Nếu bạn phải chạy Kimera trên robot với RAM 4GB, bạn sẽ
    cắt giảm module nào? Tại sao?

Q8. Kimera dùng CPU cho tất cả trừ 2D segmentation. Thiết kế
    pipeline nếu bạn có thêm 1 GPU budget cho 1 module nữa —
    module nào sẽ benefit nhiều nhất?
```

### 1.3 Hướng dẫn code

```python
# === Bài tập 1: TSDF Integration từ đầu ===
# Mục tiêu: Hiểu TSDF bằng cách implement mini version

# TODO: Implement các hàm sau
# 1. Tạo voxel grid 3D (numpy array)
# 2. Với mỗi depth image + camera pose:
#    - Project mỗi voxel vào image plane
#    - Tính signed distance = depth_measured - depth_voxel
#    - Truncate & weighted average update
# 3. Extract surface bằng marching cubes (dùng skimage)

# Thư viện cần: numpy, opencv, scikit-image
# Dataset test: Dùng 1 sequence từ TUM RGB-D dataset

# === Bài tập 2: Bag-of-Words Place Recognition ===
# Mục tiêu: Hiểu DBoW2 loop closure detection

# TODO:
# 1. Extract ORB features từ sequence ảnh
# 2. Build vocabulary tree (dùng sklearn KMeans)
# 3. Convert mỗi image thành BoW vector
# 4. Tìm similar images bằng cosine similarity
# 5. Verify bằng geometric verification (RANSAC)

# === Bài tập 3: Factor Graph với GTSAM ===
# Mục tiêu: Hiểu pose graph optimization

# pip install gtsam
# TODO:
# 1. Tạo simple pose graph: 5 poses, odometry edges + 1 loop closure
# 2. Add noise vào odometry
# 3. Optimize bằng Levenberg-Marquardt
# 4. So sánh trước/sau optimization
# 5. Thử thêm outlier loop closure → xem kết quả sai thế nào
# 6. Thêm robust cost function (Huber) → outlier bị reject không?
```

---

## 📖 Module 2: Real-time 3D Scene Graph Construction (Hydra)

### 2.1 Dàn ý kiến thức cần nắm

```
A. Kiến trúc phân tầng (5 layers)
   ├── Layer 1: Metric-Semantic 3D Mesh
   ├── Layer 2: Objects & Agents
   ├── Layer 3: Places (topological map)
   ├── Layer 4: Rooms
   ├── Layer 5: Buildings
   └── Inter-layer edges: object↔place, place↔room, room↔building

B. Active Window & Incremental Processing
   ├── Tại sao không giữ full ESDF? (memory scaling O(n))
   ├── Spatial windowing: chỉ giữ ESDF quanh robot (8m radius)
   ├── Mesh & places rời active window → pass cho frontend
   └── Constant-time processing bất kể environment size

C. Generalized Voronoi Diagram (GVD) → Places
   ├── GVD: tập voxel equidistant tới ≥2 obstacles
   ├── Brushfire algorithm trong ESDF integration
   ├── Sparsify GVD thành sparse graph of places
   ├── Node selection: basis points ≥ 4 hoặc corner template
   ├── Edge identification: flood-fill + split nếu deviate
   └── Merge nearby nodes, connect disconnected components

D. Room Detection (Community Detection approach)
   ├── Key insight: dilation đóng cửa → tách phòng
   ├── Map dilation sang graph pruning (distance < δ → remove node)
   ├── Sweep qua nhiều δ → đếm connected components
   ├── Chọn median number of rooms nr
   ├── Seeded community detection (Louvain-inspired)
   └── So sánh với Kimera: height-based vs topology-based

E. Loop Closure Detection & Scene Graph Optimization
   ├── Hierarchical descriptors: places → objects → appearance
   ├── Top-down detection: compare descriptors layer by layer
   ├── Bottom-up verification: RANSAC visual → TEASER++ objects
   ├── Embedded deformation graph optimization
   ├── Deformation graph = agent + mesh control points + places MST
   └── GNC solver cho outlier rejection + interpolation mesh
```

### 2.2 Câu hỏi sâu

```
[Câu hỏi hiểu bản chất]
Q1. GVD là "skeleton" của environment. Vẽ tay GVD cho 1 hành lang
    chữ T đơn giản. Tại sao GVD tự nhiên nằm ở giữa free space?

Q2. Giải thích tại sao dilation distance δ có thể expose rooms:
    - Vẽ ví dụ 2 phòng nối qua cửa hẹp
    - Khi δ > bề rộng cửa/2 thì chuyện gì xảy ra?
    - Tại sao dùng median thay vì max/min?

Q3. Deformation graph optimization: tại sao dùng minimum spanning
    tree của places thay vì full places graph? Trade-off gì?

Q4. Hierarchical loop closure descriptor gồm 3 level:
    - Places descriptor = histogram of distances
    - Object descriptor = histogram of labels
    - Appearance descriptor = DBoW2
    Tại sao so sánh top-down (places → objects → appearance)
    thay vì bottom-up? Khi nào top-down sẽ fail?

[Câu hỏi phân tích]
Q5. Hydra "thinking fast and slow":
    - Low-level: sensor rate (~200Hz)
    - Mid-level: 5-10Hz
    - High-level: grows with map size
    Thiết kế threading model nếu bạn có 4 cores. Cái nào share
    data với nhau? Cần lock/mutex ở đâu?

Q6. So sánh room detection:
    | Kimera (height-based)  | Hydra (topology-based) |
    Tại sao Kimera fail ở multi-floor? Hydra có weakness gì?
    (Hint: open floor plan)

[Câu hỏi mở rộng]
Q7. Hydra không detect được room label (kitchen vs bedroom).
    Đề xuất 1 phương pháp kết hợp object statistics trong room
    để gán label. Dùng LLM hay classifier? Tại sao?

Q8. Nếu robot quay lại 1 phòng đã map 10 phút trước và đồ vật
    đã thay đổi (ai đó di chuyển ghế), Hydra xử lý thế nào?
    Đề xuất cải tiến.
```

### 2.3 Hướng dẫn code

```python
# === Bài tập 1: Voronoi Diagram 2D ===
# Mục tiêu: Hiểu GVD trước khi lên 3D

# TODO:
# 1. Tạo occupancy grid 2D (numpy), vẽ walls & doors
# 2. Compute distance transform (scipy.ndimage.distance_transform_edt)
# 3. Tìm GVD: voxels có ≥2 nearest obstacles khác nhau
#    (Hint: dùng distance_transform_edt với return_indices=True)
# 4. Sparsify GVD thành graph (networkx)
# 5. Visualize: map + GVD + sparse graph overlay

# === Bài tập 2: Room Segmentation by Dilation ===
# Mục tiêu: Implement room detection algorithm

# TODO:
# 1. Từ bài tập 1, lấy sparse graph of places
# 2. Mỗi node có distance to nearest wall
# 3. Sweep δ từ 0.3m → 1.5m, step 0.1m
# 4. Với mỗi δ: prune nodes (distance < δ), đếm connected components
# 5. Lấy median number of components → chọn δ*
# 6. Dùng seeded community detection assign unlabeled nodes
# 7. Visualize rooms bằng màu khác nhau

# Thư viện: numpy, scipy, networkx, matplotlib
# Bonus: thử với floor plan thật (download từ internet)

# === Bài tập 3: Scene Graph Data Structure ===
# Mục tiêu: Thiết kế data structure cho 3D Scene Graph

# TODO: Implement class hierarchy
class SceneGraphNode:
    # id, position, attributes, layer, edges_within, edges_across
    pass

class ObjectNode(SceneGraphNode):
    # semantic_label, bounding_box, centroid, mesh_vertex_ids
    pass

class PlaceNode(SceneGraphNode):
    # obstacle_distance, nearest_mesh_vertex
    pass

class RoomNode(SceneGraphNode):
    # place_ids, room_label (optional)
    pass

class SceneGraph:
    # layers: dict[int, list[Node]]
    # intra_edges: edges within same layer
    # inter_edges: edges across layers
    # methods: add_node, add_edge, query_by_room, query_nearby_objects
    pass

# Test: tạo mini scene graph cho 1 apartment 3 phòng, 5 objects
# Query: "tìm tất cả objects trong kitchen"
# Query: "tìm đường từ bedroom đến kitchen qua places"
```

---

## 📖 Module 3: LLM-based Task Planning trên Scene Graphs (SayPlan)

### 3.1 Dàn ý kiến thức cần nắm

```
A. Vấn đề Scalability
   ├── LLM context window có giới hạn token
   ├── Full 3DSG lớn → vượt token limit
   ├── Ví dụ: Office 6731 tokens full → 878 tokens collapsed (87%)
   └── Cần: chỉ đưa LLM thông tin relevant

B. 3D Scene Graph Representation cho LLM
   ├── JSON serialization của NetworkX graph
   ├── Node format: name, type, location, affordances, state, attributes
   ├── Edge format: room↔object
   ├── Hierarchy: Floor → Room → Asset → Object
   └── Agent node: robot location

C. Stage 1: Semantic Search
   ├── collapse(G): chỉ expose top level (floors)
   ├── LLM dùng expand(node) / contract(node) API
   ├── Chain-of-thought prompting guide LLM explore
   ├── Memory: list đã expand để tránh lặp
   ├── Markovian: không cần full interaction history
   └── Terminate khi tìm đủ relevant items

D. Stage 2: Iterative Replanning
   ├── LLM generate high-level plan (manipulation actions)
   ├── Path planner (Dijkstra) handle navigation
   ├── Scene graph simulator verify plan
   ├── Feedback: text mô tả lỗi ("cannot pick banana")
   ├── LLM nhận feedback → sửa plan → verify lại
   └── Loop cho đến "success" hoặc max iterations (5)

E. Prompt Engineering
   ├── Agent Role definition
   ├── Environment Functions: goto, access, pickup, release, open/close...
   ├── Environment State: ontop_of, inside_of, inside_hand...
   ├── Output format: JSON với chain_of_thought, reasoning, command
   └── In-context examples (few-shot)
```

### 3.2 Câu hỏi sâu

```
[Câu hỏi hiểu bản chất]
Q1. Tại sao collapse graph trước rồi mới semantic search, thay vì
    đưa full graph cho LLM và bảo "chỉ focus vào phần relevant"?
    (Hint: attention mechanism, token cost, hallucination)

Q2. SayPlan dùng Memory = list of expanded nodes. Tại sao cần
    Markovian approach? Nếu giữ full conversation history thì
    token cost tăng thế nào theo số bước search?

Q3. Iterative replanning: tại sao tách navigation (Dijkstra) ra
    khỏi LLM? LLM có thể plan navigation không? Trade-off?

Q4. Scene graph simulator feedback là text. Thiết kế feedback
    message cho các failure case sau:
    - Robot cố pickup khi tay đang cầm đồ
    - Robot cố open fridge đã open sẵn
    - Robot cố goto room không tồn tại (hallucinated)

[Câu hỏi phân tích]
Q5. SayPlan fail ở negation ("find office without cabinet").
    Phân tích tại sao LLM khó xử lý negation trên graph.
    Đề xuất giải pháp (graph query language? explicit filter?).

Q6. Token budget analysis:
    - Prompt template ≈ 3900 tokens (cố định)
    - GPT-4 context = 8192 tokens
    - Còn lại cho 3DSG + response
    Tính: với Office (collapsed = 878 tokens), tối đa expand
    bao nhiêu rooms trước khi hết token? (mean room ≈ 27 tokens)

Q7. SayPlan giả sử 3DSG đã có sẵn. Nếu robot vừa explore
    vừa plan (Hydra + SayPlan), có conflict gì không?
    - Scene graph đang thay đổi trong khi LLM đang plan
    - Object mới xuất hiện giữa chừng
    Đề xuất synchronization strategy.

[Câu hỏi thiết kế]
Q8. Thiết kế evaluation metric mới cho task planning:
    - Correctness & Executability chưa đủ
    - Cần đo: efficiency (số bước), token cost, replanning rounds
    - Làm sao đo "common sense quality" của plan?
```

### 3.3 Hướng dẫn code

```python
# === Bài tập 1: Scene Graph JSON & Collapse/Expand ===
# Mục tiêu: Implement core SayPlan graph operations

# TODO:
# 1. Tạo mini 3DSG bằng dict/JSON:
#    Floor1 → [kitchen, bedroom, bathroom]
#    kitchen → [fridge(closed), table] 
#    fridge → [apple, milk]
#    bedroom → [bed, desk] → desk → [laptop, book]
#
# 2. Implement:
#    - collapse(G): return graph chỉ có floor nodes
#    - expand(node_name): reveal children
#    - contract(node_name): hide children
#    - token_count(G): estimate tokens (đếm chars/4)
#
# 3. Test: simulate LLM search cho "find the apple"
#    Log: mỗi bước in graph state + token count

# === Bài tập 2: Scene Graph Simulator ===
# Mục tiêu: Implement verify_plan()

# TODO:
# 1. Define world state: agent location, object states, holding
# 2. Implement action executor:
#    - goto(room): update agent location
#    - pickup(obj): check accessible? check not holding? update
#    - open(asset): check at location? check closed?
#    - release(obj): check holding? check at asset?
# 3. verify_plan(plan): execute step by step
#    - Nếu fail → return text feedback
#    - Nếu success → return "success"
# 4. Test với plan đúng và plan sai (thiếu open fridge trước pickup)

# === Bài tập 3: LLM Planning Pipeline (dùng API thật) ===
# Mục tiêu: Kết nối LLM với scene graph

# Cần: OpenAI API key hoặc Anthropic API key
# TODO:
# 1. Thiết kế prompt template (copy style từ SayPlan Appendix J)
# 2. Serialize mini 3DSG thành JSON string
# 3. Gửi tới LLM: instruction + collapsed graph
# 4. Parse LLM response (JSON) → extract command
# 5. Execute command trên graph → update graph → gửi lại LLM
# 6. Loop cho tới terminate

# Stretch: thêm iterative replanning
# 1. LLM output plan
# 2. verify_plan() → feedback
# 3. Nếu fail: append feedback vào prompt → LLM sửa plan
# 4. Repeat max 5 lần

# QUAN TRỌNG: Log mọi thứ — token count mỗi bước, plan changes
```

---

## 📖 Module 4: Open-Vocabulary Scene Graphs (Open3DSG)

### 4.1 Dàn ý kiến thức cần nắm

```
A. Vấn đề với Closed-Vocabulary
   ├── 3DSSG dataset: 160 objects, 27 predicates → quá hẹp
   ├── Tail-class performance rất thấp (supervised methods)
   ├── Downstream tasks cần vocabulary rộng hơn training set
   └── Mục tiêu: predict object + relationship không giới hạn label

B. Tại sao CLIP không đủ cho Relationships?
   ├── CLIP = contrastive learning → bag-of-words behavior
   ├── "Dog chasing cat" ≈ "Cat chasing dog" trong CLIP space
   ├── Compositionality benchmarks: CLIP fail nặng
   ├── NegCLIP cải thiện không đáng kể
   └── Cần generative model (LLM) cho compositional reasoning

C. Architecture: 2-step Distillation
   ├── Step 1: Extract 2D features
   │   ├── Object features: OpenSeg (pixel-wise) > CLIP (global)
   │   ├── Relationship features: InstructBLIP image encoder
   │   ├── Frame selection: visibility check + bounding box area
   │   └── Multi-view fusion: average pooling across top-k frames
   │
   ├── Step 2: Distill vào 3D GNN
   │   ├── Input: point cloud P + instance masks M
   │   ├── PointNet extract node/edge features
   │   ├── GNN message passing refine features
   │   ├── Loss = cosine similarity(2D features, 3D features)
   │   └── Sau training: 3D features ≈ 2D VLM features

D. Inference: 2-step Prediction
   ├── Node prediction: cosine_sim(CLIP_text, node_feature) → argmax
   ├── Edge prediction: InstructBLIP Qformer + LLM
   │   ├── Input: edge feature (thay ViT output) + object names
   │   ├── Prompt: "Describe relationship between [obj1] and [obj2]?"
   │   └── Output: free-form relationship text
   └── Label mapping: BERT encode LLM output → cosine sim với label set

E. 2D-3D Feature Fusion
   ├── Nếu có 2D images → average pool 2D + 3D features
   ├── 2D tốt cho small objects, 3D tốt cho large distinctive shapes
   └── 3D-only mode: khi không có posed images at test time
```

### 4.2 Câu hỏi sâu

```
[Câu hỏi hiểu bản chất]
Q1. Tại sao dùng OpenSeg thay vì CLIP cho object features?
    - OpenSeg: pixel-wise embeddings
    - CLIP: global image/crop embedding
    Trong trường hợp nào CLIP vẫn tốt hơn?

Q2. Cosine similarity loss cho distillation:
    L = 1 - cos(F_2D, F_3D)
    Tại sao dùng cosine thay vì MSE? Liên hệ tới metric learning.

Q3. InstructBLIP pipeline cho relationship:
    ViT encoder → Qformer → LLM
    Open3DSG thay ViT output bằng 3D GNN edge features.
    Điều này có nghĩa Qformer phải bridge từ distribution khác.
    Tại sao vẫn hoạt động? (Hint: distillation loss align spaces)

Q4. Label mapping dùng BERT thay vì CLIP. Tại sao?
    - BERT: bidirectional, good word embeddings
    - CLIP text encoder: contrastive with images
    Khi nào dùng CLIP text, khi nào dùng BERT cho text matching?

[Câu hỏi phân tích]
Q5. Bảng 2 (frequency-based evaluation):
    - Supervised methods: Head=0.92, Tail=0.24 (objects)
    - Open3DSG:          Head=0.60, Tail=0.42 (objects)
    Phân tích: tại sao open-vocab tốt hơn ở tail nhưng kém ở head?
    Có thể kết hợp 2 approaches không?

Q6. Open3DSG train trên ScanNet nhưng test trên 3DSSG.
    Tại sao không train trên 3DSSG luôn?
    (Hint: portrait vs landscape FOV, frame quality)
    Đây có phải domain gap không? Ảnh hưởng thế nào?

Q7. Long-distance relationships (Supplementary Fig. C):
    2D-only methods cần 2 objects cùng visible trong 1 frame.
    3D method thì không. Nhưng 3D features cho distant objects
    có thực sự encode relationship info? Hay chỉ dựa vào LLM?

[Câu hỏi thiết kế]  
Q8. Thiết kế evaluation protocol cho open-vocabulary scene graphs:
    - Closed-vocab recall không đủ
    - Human evaluation quá expensive
    - Đề xuất: automated metric dùng LLM-as-judge?
    - Cần đo gì? (correctness, specificity, diversity, consistency)
```

### 4.3 Hướng dẫn code

```python
# === Bài tập 1: CLIP Zero-shot Classification ===
# Mục tiêu: Hiểu CLIP embedding space

# pip install transformers torch
# TODO:
# 1. Load CLIP model (openai/clip-vit-base-patch32)
# 2. Encode text queries: ["chair", "table", "bed", "floor", "wall"]
# 3. Encode 1 image chứa furniture
# 4. Compute cosine similarity → classify
# 5. THỬ RELATIONSHIP: encode "chair on floor" vs "floor on chair"
#    → similarity gần nhau? → chứng minh CLIP thiếu compositionality

# === Bài tập 2: PointNet Feature Extraction ===
# Mục tiêu: Hiểu 3D point cloud feature learning

# pip install torch torch-geometric
# TODO:
# 1. Implement mini PointNet:
#    - Input: N x 3 point cloud
#    - MLP per point: 3 → 64 → 128 → 1024
#    - Max pooling → global feature 1024-dim
# 2. Train trên ModelNet10 (classify 10 shapes)
# 3. Visualize learned features bằng t-SNE
# 4. Thử: 2 objects gần nhau → concat point clouds + mask
#    → extract "relationship feature"

# === Bài tập 3: Feature Distillation Pipeline ===
# Mục tiêu: Distill 2D features vào 3D network

# TODO (simplified version):
# 1. Dataset: ScanNet 1 scene, posed RGB-D images
# 2. Cho mỗi object instance:
#    a. Select top-5 frames (visibility check)
#    b. Extract CLIP features từ crops → average → F_2D
#    c. Extract point cloud → PointNet → F_3D
# 3. Train: minimize 1 - cosine_sim(F_2D, F_3D)
# 4. Test: query objects bằng text
#    - Encode text bằng CLIP text encoder
#    - cosine_sim(text_feature, F_3D) → classify

# Simplified: nếu không có ScanNet, dùng synthetic data
# Tạo colored point clouds + render images → distill

# === Bài tập 4: LLM Relationship Prediction ===
# Mục tiêu: Dùng LLM predict relationships

# TODO:
# 1. Cho danh sách object pairs:
#    [("tv", "wall"), ("pillow", "bed"), ("chair", "desk")]
# 2. Prompt LLM: "Describe the spatial relationship between {obj1}
#    and {obj2} in an indoor scene. Answer with 2-4 words."
# 3. Map output → fixed label set (27 labels từ 3DSSG)
#    dùng sentence-transformers cosine similarity
# 4. So sánh: CLIP zero-shot vs LLM generative approach
#    - Accuracy trên 20 test pairs
```

---

## 🔗 Module 5: Kết nối tất cả — Research Gaps & Project Ideas

### 5.1 Bản đồ quan hệ giữa 4 papers

```
                    ┌─────────────┐
                    │   Open3DSG  │ Open-vocab labels
                    │  (CVPR'24)  │ & relationships
                    └──────┬──────┘
                           │ provides open-vocab
                           │ node/edge labels
                           ▼
┌──────────┐    builds    ┌─────────────┐    grounds    ┌──────────┐
│  Kimera  │───────────►  │    Hydra    │──────────────►│ SayPlan  │
│(ICRA'20) │  foundation  │  (RSS'22)   │  scene graph  │(CoRL'23) │
│ VIO+SLAM │              │ real-time   │  for LLM      │ LLM plan │
└──────────┘              │ scene graph │  planning     └──────────┘
                          └─────────────┘

Gap: Không ai kết nối cả 3 mũi tên thành 1 system!
```

### 5.2 Câu hỏi nghiên cứu mở

```
RQ1. Có thể chạy Open3DSG online (incremental) thay vì batch
     trên complete point cloud không? Bottleneck ở đâu?

RQ2. SayPlan dùng fixed-vocabulary scene graph. Nếu thay bằng
     open-vocabulary (Open3DSG), LLM có plan tốt hơn không?
     Hay noise từ open-vocab predictions gây hại?

RQ3. Hydra detect rooms nhưng không label. Open3DSG classify
     objects. Kết hợp: dùng object composition trong room
     + LLM → auto label rooms. Feasible? Accurate?

RQ4. Current systems: map trước → plan sau.
     Có thể interleave: explore + build scene graph + plan
     simultaneously không? Active exploration guided by task?

RQ5. Evaluation crisis: không có benchmark tốt cho open-vocab
     scene graphs cho task planning. Thiết kế benchmark:
     - Environment diversity
     - Task complexity levels  
     - Metrics beyond correctness/executability
```

### 5.3 Mini project ideas (2-4 tuần mỗi cái)

```
[Beginner]
P1. Implement room labeling cho Hydra output
    - Input: scene graph với rooms (unlabeled) + objects (labeled)
    - Method: aggregate object labels per room → prompt LLM
    - Output: room labels (kitchen, bedroom, etc.)
    - Eval: trên uHumans2 dataset

[Intermediate]
P2. SayPlan trên dynamic scene graph
    - Modify SayPlan pipeline: graph có thể thay đổi giữa search/plan
    - Implement "graph diff" notification cho LLM
    - Eval: planning performance khi objects move

[Advanced]
P3. Incremental Open3DSG
    - Stream point cloud patches (simulate robot exploration)
    - Update GNN features incrementally
    - Benchmark: latency vs batch accuracy trade-off
```

---

## 📦 Setup & Resources

### Repositories

```
Hydra:      https://github.com/MIT-SPARK/Hydra
Kimera:     https://github.com/MIT-SPARK/Kimera
GTSAM:      https://github.com/borglab/gtsam
Voxblox:    https://github.com/ethz-asl/voxblox
Open3DSG:   https://kochsebastian.com/open3dsg
SayPlan:    https://sayplan.github.io
```

### Datasets

```
EuRoC:      https://projects.asl.ethz.ch/datasets/doku.php?id=kmavvisualinertialdatasets
ScanNet:    http://www.scan-net.org/
3DSSG:      https://3dssg.github.io/
TUM RGB-D:  https://vision.in.tum.de/data/datasets/rgbd-dataset
```

### Thư viện Python cần cài

```bash
pip install numpy scipy networkx matplotlib scikit-image
pip install torch torchvision
pip install transformers  # CLIP, BERT
pip install gtsam         # factor graphs
pip install open3d        # 3D visualization
pip install pdfplumber    # nếu cần đọc paper PDFs
```

---

## ✅ Checklist tự đánh giá

```
Module 1 (Kimera):
[ ] Giải thích được IMU preintegration
[ ] Vẽ được factor graph cho VIO
[ ] Implement được TSDF integration
[ ] Hiểu PCM outlier rejection

Module 2 (Hydra):
[ ] Vẽ được GVD cho floor plan đơn giản
[ ] Implement được room detection by dilation
[ ] Giải thích deformation graph optimization
[ ] So sánh được Hydra vs Kimera room detection

Module 3 (SayPlan):
[ ] Implement được collapse/expand graph operations
[ ] Viết được scene graph simulator
[ ] Chạy được LLM planning pipeline end-to-end
[ ] Phân tích được token budget

Module 4 (Open3DSG):
[ ] Chứng minh được CLIP thiếu compositionality
[ ] Implement được feature distillation (simplified)
[ ] So sánh CLIP vs LLM cho relationship prediction
[ ] Giải thích được tại sao open-vocab tốt ở tail classes

Tổng hợp:
[ ] Vẽ được bản đồ quan hệ 4 papers
[ ] Xác định được ≥ 2 research gaps
[ ] Hoàn thành ≥ 1 mini project
```
