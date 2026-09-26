# Part 34: Graph เบื้องต้น: Representation, BFS, DFS

> ขั้นตอนที่ 331-340 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. Graph คืออะไร ศัพท์พื้นฐาน
2. การแทน Graph: Adjacency Matrix vs Adjacency List
3. Directed vs Undirected Graph
4. Weighted vs Unweighted Graph
5. Depth-First Search (DFS) บน Graph
6. Breadth-First Search (BFS) บน Graph
7. การหา Shortest Path ด้วย BFS (Unweighted Graph)
8. การตรวจหา Cycle ใน Graph
9. Topological Sort เบื้องต้น
10. แบบฝึกหัดและสรุป

---

## 1. Graph คืออะไร ศัพท์พื้นฐาน

**Graph** คือโครงสร้างข้อมูลที่ประกอบด้วย **vertex/node** (จุด) และ **edge**
(เส้นเชื่อม) ระหว่างจุดเหล่านั้น — ทั่วไปกว่า Tree มาก (Tree คือ Graph ชนิด
พิเศษที่ไม่มี cycle และมี root ชัดเจน) ใช้จำลองความสัมพันธ์ในโลกจริงได้กว้างขวาง
มาก เช่น เครือข่ายสังคม, แผนที่ถนน, ระบบแนะนำสินค้า

```
ศัพท์พื้นฐาน:

    A --- B         Vertex (จุด): A, B, C, D
    |     |         Edge (เส้นเชื่อม): A-B, A-C, B-D, C-D
    C --- D         Degree ของ A = 2 (จำนวน edge ที่เชื่อมกับ A)
                     Path: A -> B -> D (ลำดับ vertex ที่เชื่อมต่อกัน)
                     Cycle: A -> B -> D -> C -> A (path ที่กลับมาจุดเริ่มต้น)
```

## 2. การแทน Graph: Adjacency Matrix vs Adjacency List

### Adjacency Matrix: ตาราง 2 มิติ

```java
public class AdjacencyMatrixDemo {
    public static void main(String[] args) {
        int n = 4; // จำนวน vertex (0, 1, 2, 3)
        int[][] matrix = new int[n][n];

        // เพิ่ม edge: 0-1, 0-2, 1-3, 2-3
        matrix[0][1] = matrix[1][0] = 1; // undirected graph: เชื่อมสองทาง
        matrix[0][2] = matrix[2][0] = 1;
        matrix[1][3] = matrix[3][1] = 1;
        matrix[2][3] = matrix[3][2] = 1;

        for (int[] row : matrix) {
            System.out.println(java.util.Arrays.toString(row));
        }
        // matrix[i][j] == 1 หมายถึงมี edge ระหว่าง vertex i และ j
    }
}
```

**ข้อดี**: เช็คว่ามี edge ระหว่างสอง vertex หรือไม่ได้ทันที O(1) | **ข้อเสีย**:
ใช้หน่วยความจำ O(V²) เสมอ แม้ graph มี edge น้อยมาก (sparse graph) — สิ้นเปลือง
มากสำหรับ graph ขนาดใหญ่

### Adjacency List: List ของ neighbor แต่ละ vertex (นิยมมากกว่าในทางปฏิบัติ)

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.HashMap;

public class AdjacencyListDemo {
    public static void main(String[] args) {
        Map<Integer, List<Integer>> graph = new HashMap<>();

        // เพิ่ม vertex และ edge
        graph.put(0, new ArrayList<>(List.of(1, 2)));
        graph.put(1, new ArrayList<>(List.of(0, 3)));
        graph.put(2, new ArrayList<>(List.of(0, 3)));
        graph.put(3, new ArrayList<>(List.of(1, 2)));

        System.out.println(graph);
        // {0=[1, 2], 1=[0, 3], 2=[0, 3], 3=[1, 2]}
        // vertex 0 เชื่อมกับ 1 และ 2 เท่านั้น (ไม่ต้องเก็บว่าไม่เชื่อมกับตัวอื่นเลย - ประหยัดกว่า)
    }
}
```

**ข้อดี**: ใช้หน่วยความจำ O(V + E) — ประหยัดกว่ามากสำหรับ **sparse graph**
(graph ที่มี edge น้อยเทียบกับจำนวน vertex ที่เป็นไปได้ทั้งหมด ซึ่งเป็นกรณีที่
พบบ่อยที่สุดในโลกจริง เช่น เครือข่ายสังคม) | **ข้อเสีย**: เช็คว่ามี edge ระหว่าง
สอง vertex หรือไม่ ต้องไล่หาใน list (O(degree) ไม่ใช่ O(1))

**หลักปฏิบัติ**: ใช้ **Adjacency List เป็นค่าเริ่มต้นเสมอ** ในโค้ดจริง เว้นแต่
graph มีขนาดเล็กมากหรือมี edge หนาแน่นมาก (dense graph)

## 3. Directed vs Undirected Graph

```java
import java.util.List;
import java.util.Map;
import java.util.ArrayList;
import java.util.HashMap;

public class DirectedGraphDemo {
    public static void main(String[] args) {
        // Undirected: A-B หมายถึงเดินทางได้ทั้งสองทาง (ทบทวนจากตัวอย่างก่อนหน้า)

        // Directed Graph: A->B ไม่ได้แปลว่า B->A ได้ (เช่น "ติดตาม" ใน Twitter)
        Map<String, List<String>> directedGraph = new HashMap<>();
        directedGraph.put("Alice", new ArrayList<>(List.of("Bob")));   // Alice ติดตาม Bob
        directedGraph.put("Bob", new ArrayList<>(List.of("Charlie")));  // Bob ติดตาม Charlie
        directedGraph.put("Charlie", new ArrayList<>());                 // Charlie ไม่ติดตามใคร

        // สังเกต: Bob ไม่ได้ติดตาม Alice กลับ (Alice->Bob เป็น edge ทางเดียว)
        System.out.println(directedGraph);
    }
}
```

**ตัวอย่างในโลกจริง**:
- **Undirected**: เพื่อนใน Facebook (ถ้า A เป็นเพื่อน B, B ก็เป็นเพื่อน A เสมอ),
  ถนนสองทาง
- **Directed**: ผู้ติดตามใน Twitter/Instagram, เว็บไซต์ที่ลิงก์ไปยังเว็บอื่น,
  ถนนทางเดียว, ความสัมพันธ์ "งานก่อนหน้า" (prerequisite) ของวิชาเรียน

## 4. Weighted vs Unweighted Graph

**Weighted Graph** มี**ค่าน้ำหนัก (weight/cost)** บนแต่ละ edge (เช่น ระยะทาง,
ราคา, เวลา) — ใช้บ่อยในปัญหา shortest path ที่ซับซ้อนกว่าการนับจำนวน edge
(Dijkstra's Algorithm ซึ่งเป็นเนื้อหาระดับสูงกว่าที่จะกล่าวถึงในภาพรวม)

```java
public class WeightedGraphDemo {
    record Edge(String destination, int weight) { }

    public static void main(String[] args) {
        Map<String, List<Edge>> weightedGraph = new HashMap<>();
        weightedGraph.put("Bangkok", List.of(
            new Edge("ChiangMai", 700),  // 700 กม.
            new Edge("Phuket", 850)
        ));
        weightedGraph.put("ChiangMai", List.of(new Edge("Bangkok", 700)));
        weightedGraph.put("Phuket", List.of(new Edge("Bangkok", 850)));

        System.out.println(weightedGraph.get("Bangkok"));
    }

    static Map<String, List<Edge>> Map; // (import java.util.Map, List, HashMap ต้องเพิ่มจริงตอน compile)
}
```
(หมายเหตุ: ตัวอย่างข้างบนเพื่ออธิบายแนวคิด — ในโค้ดจริงต้อง `import java.util.*`
ให้ครบและจัดโครงสร้างเป็น class ที่ compile ได้สมบูรณ์)

## 5. Depth-First Search (DFS) บน Graph

DFS บน Graph ใช้แนวคิดเดียวกับ Tree (Part 33) แต่ **ต้องมี `visited` set** เพื่อ
ป้องกันการวน loop ไม่จบ (เพราะ Graph มี cycle ได้ ต่างจาก Tree)

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class GraphDFSDemo {
    static Map<String, List<String>> graph = new HashMap<>();

    static void dfs(String node, Set<String> visited) {
        if (visited.contains(node)) return; // ป้องกันการเยี่ยม node เดิมซ้ำ (สำคัญมาก!)

        visited.add(node);
        System.out.print(node + " ");

        for (String neighbor : graph.getOrDefault(node, new ArrayList<>())) {
            dfs(neighbor, visited); // recursive call ไปยัง neighbor แต่ละตัว
        }
    }

    public static void main(String[] args) {
        graph.put("A", List.of("B", "C"));
        graph.put("B", List.of("A", "D"));
        graph.put("C", List.of("A", "D"));
        graph.put("D", List.of("B", "C"));

        dfs("A", new HashSet<>()); // A B D C (ลงลึกไปทางหนึ่งก่อนจนสุด แล้วถอยกลับมาทางอื่น)
    }
}
```

**ทำไมต้องมี `visited` set**: ถ้าไม่มี การเยี่ยม A -> B -> A -> B -> ... จะวน
ไม่จบเพราะ B เชื่อมกลับมาที่ A (cycle) — นี่คือความแตกต่างสำคัญจาก Tree
traversal ที่ไม่มี cycle จึงไม่ต้องกัน

## 6. Breadth-First Search (BFS) บน Graph

ทบทวนแนวคิดจาก Part 25 (ตัวอย่าง preview) และ Part 33 — BFS ใช้ Queue เยี่ยม
ทีละระดับ เหมาะกับการหา**เส้นทางที่สั้นที่สุด**ใน unweighted graph

```java
import java.util.ArrayDeque;
import java.util.HashSet;
import java.util.Queue;
import java.util.Set;

public class GraphBFSDemo {
    static java.util.Map<String, java.util.List<String>> graph = new java.util.HashMap<>();

    static void bfs(String start) {
        Queue<String> queue = new ArrayDeque<>();
        Set<String> visited = new HashSet<>();

        queue.offer(start);
        visited.add(start);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            System.out.print(current + " ");

            for (String neighbor : graph.getOrDefault(current, new java.util.ArrayList<>())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor); // สำคัญ: mark visited "ตอนใส่ queue" ไม่ใช่ตอน poll ออกมา
                    queue.offer(neighbor); // ป้องกันไม่ให้ node เดียวกันถูกใส่ queue ซ้ำหลายครั้ง
                }
            }
        }
    }

    public static void main(String[] args) {
        graph.put("A", java.util.List.of("B", "C"));
        graph.put("B", java.util.List.of("A", "D"));
        graph.put("C", java.util.List.of("A", "D"));
        graph.put("D", java.util.List.of("B", "C"));

        bfs("A"); // A B C D (เยี่ยมทีละระดับจากจุดเริ่มต้น)
    }
}
```

**ข้อควรระวังสำคัญ**: ต้อง mark `visited` **ทันทีที่ใส่เข้า queue** (ไม่ใช่รอ
ตอน poll ออกมาแล้วค่อย mark) เพื่อป้องกันไม่ให้ node เดียวกันถูกใส่เข้า queue
ซ้ำหลายครั้งโดยไม่จำเป็น

## 7. การหา Shortest Path ด้วย BFS (Unweighted Graph)

```java
import java.util.ArrayDeque;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Queue;
import java.util.Set;

public class ShortestPathBFSDemo {
    static Map<String, List<String>> graph = new HashMap<>();

    static int shortestPath(String start, String target) {
        if (start.equals(target)) return 0;

        Queue<String> queue = new ArrayDeque<>();
        Map<String, Integer> distance = new HashMap<>();
        Set<String> visited = new HashSet<>();

        queue.offer(start);
        visited.add(start);
        distance.put(start, 0);

        while (!queue.isEmpty()) {
            String current = queue.poll();

            for (String neighbor : graph.getOrDefault(current, List.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    distance.put(neighbor, distance.get(current) + 1);

                    if (neighbor.equals(target)) {
                        return distance.get(neighbor); // เจอเป้าหมายแล้ว คืนระยะทางทันที
                    }
                    queue.offer(neighbor);
                }
            }
        }
        return -1; // ไม่มีเส้นทางไปถึงเป้าหมาย
    }

    public static void main(String[] args) {
        graph.put("A", List.of("B", "C"));
        graph.put("B", List.of("A", "D"));
        graph.put("C", List.of("A", "D"));
        graph.put("D", List.of("B", "C", "E"));
        graph.put("E", List.of("D"));

        System.out.println(shortestPath("A", "E")); // 3 (A -> B/C -> D -> E)
    }
}
```

**ทำไม BFS หา shortest path ได้แม่นยำ (สำหรับ unweighted graph)**: เพราะ BFS
เยี่ยม node **ทีละระดับ** — ระดับที่ 1 คือ node ที่ห่างจาก start 1 ก้าว, ระดับ
ที่ 2 คือห่าง 2 ก้าว ฯลฯ ดังนั้น node แรกที่เจอเป้าหมายจะมาจากเส้นทางที่สั้น
ที่สุดเสมอ (**DFS ทำแบบนี้ไม่ได้** เพราะลงลึกทางหนึ่งก่อน อาจเจอเส้นทางที่ยาว
กว่าก่อนเส้นทางที่สั้นกว่า)

## 8. การตรวจหา Cycle ใน Graph

```java
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class CycleDetectionDemo {
    static Map<Integer, List<Integer>> graph = new HashMap<>();

    // สำหรับ undirected graph: ถ้าเจอ neighbor ที่ visited แล้ว "และไม่ใช่ parent" แสดงว่ามี cycle
    static boolean hasCycle(int node, int parent, Set<Integer> visited) {
        visited.add(node);

        for (int neighbor : graph.getOrDefault(node, List.of())) {
            if (!visited.contains(neighbor)) {
                if (hasCycle(neighbor, node, visited)) return true;
            } else if (neighbor != parent) { // visited แล้ว และไม่ใช่ parent -> เจอ cycle!
                return true;
            }
        }
        return false;
    }

    public static void main(String[] args) {
        graph.put(0, List.of(1, 2));
        graph.put(1, List.of(0, 2));
        graph.put(2, List.of(0, 1)); // 0-1-2-0 คือ cycle

        System.out.println(hasCycle(0, -1, new HashSet<>())); // true
    }
}
```

## 9. Topological Sort เบื้องต้น

**Topological Sort** ใช้กับ **Directed Acyclic Graph (DAG)** เท่านั้น (directed
graph ที่ไม่มี cycle) — จัดเรียง vertex ให้ **ทุก edge ชี้จากซ้ายไปขวาเสมอ**
ใช้แก้ปัญหา "ลำดับการทำงานที่มี dependency" เช่น การเรียงลำดับวิชาเรียนตาม
prerequisite

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Queue;

public class TopologicalSortDemo {
    public static void main(String[] args) {
        // Prerequisite graph: คีย์ต้องเรียนก่อน จึงจะเรียนค่าที่ตามมาได้
        Map<String, List<String>> prerequisites = new HashMap<>();
        prerequisites.put("Calculus1", List.of("Calculus2"));
        prerequisites.put("Calculus2", List.of("Physics"));
        prerequisites.put("Programming1", List.of("Programming2"));
        prerequisites.put("Programming2", List.of());
        prerequisites.put("Physics", List.of());

        // ใช้ Kahn's Algorithm: นับ in-degree (จำนวน edge ที่ชี้เข้ามา) ของแต่ละ vertex
        Map<String, Integer> inDegree = new HashMap<>();
        for (String course : prerequisites.keySet()) {
            inDegree.putIfAbsent(course, 0);
            for (String next : prerequisites.get(course)) {
                inDegree.merge(next, 1, Integer::sum);
            }
        }

        Queue<String> queue = new ArrayDeque<>();
        for (Map.Entry<String, Integer> entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) queue.offer(entry.getKey()); // เริ่มจาก vertex ที่ไม่มี prerequisite
        }

        List<String> order = new ArrayList<>();
        while (!queue.isEmpty()) {
            String course = queue.poll();
            order.add(course);

            for (String next : prerequisites.getOrDefault(course, List.of())) {
                inDegree.merge(next, -1, Integer::sum);
                if (inDegree.get(next) == 0) queue.offer(next);
            }
        }

        System.out.println("ลำดับการเรียนที่ถูกต้อง: " + order);
        // เช่น: [Calculus1, Programming1, Calculus2, Programming2, Physics] (ลำดับอาจต่างกันได้บ้าง)
    }
}
```

**ตัวอย่างการใช้งานจริงของ Topological Sort**: การจัดลำดับ build dependency
ใน Maven/Gradle (Part 61-62), การจัดลำดับ task ใน CI/CD pipeline (Part 97),
การจัดลำดับวิชาเรียนตาม prerequisite, Task Scheduler ที่มี dependency ระหว่างกัน

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `countConnectedComponents(graph, n)` ที่นับจำนวนกลุ่มที่แยก
จากกัน (connected components) ใน undirected graph

**เฉลย:**

```java
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class Exercise1 {
    static int countConnectedComponents(Map<Integer, List<Integer>> graph, int n) {
        Set<Integer> visited = new HashSet<>();
        int count = 0;

        for (int i = 0; i < n; i++) {
            if (!visited.contains(i)) {
                count++;
                dfs(graph, i, visited);
            }
        }
        return count;
    }

    static void dfs(Map<Integer, List<Integer>> graph, int node, Set<Integer> visited) {
        visited.add(node);
        for (int neighbor : graph.getOrDefault(node, List.of())) {
            if (!visited.contains(neighbor)) {
                dfs(graph, neighbor, visited);
            }
        }
    }
}
```

**2)** อธิบายว่าทำไม Adjacency List เหมาะกับ sparse graph มากกว่า Adjacency
Matrix

**เฉลย**: Adjacency Matrix ใช้หน่วยความจำ O(V²) เสมอ ไม่ว่า graph จะมี edge
กี่ตัว เพราะต้องเตรียมช่องสำหรับทุกคู่ vertex ที่เป็นไปได้ ในขณะที่ Adjacency
List ใช้หน่วยความจำ O(V+E) — ถ้า graph เป็น sparse (E << V²) Adjacency List
จะประหยัดหน่วยความจำกว่ามาก เช่น social network ที่มีผู้ใช้ล้านคน แต่แต่ละคน
มีเพื่อนเฉลี่ยแค่ 100 คน การใช้ Adjacency Matrix จะต้องใช้หน่วยความจำ 10^12 ช่อง
ในขณะที่ Adjacency List ใช้แค่ประมาณ 10^8 ช่องเท่านั้น

**3)** ใช้ BFS เขียนโปรแกรมตรวจสอบว่า graph เป็น **Bipartite** หรือไม่ (แบ่ง
vertex ออกเป็น 2 กลุ่มได้ โดยที่ edge ทุกตัวเชื่อมระหว่างกลุ่มต่างกันเท่านั้น)

**เฉลย**: ใช้ BFS ระบายสี vertex สลับกัน (0 หรือ 1) ทีละระดับ ถ้าพบ neighbor
ที่มีสีเดียวกับตัวเองแสดงว่าไม่ใช่ bipartite — เป็นการประยุกต์ BFS ที่พบบ่อยใน
โจทย์ graph coloring

### สรุปเนื้อหา Part 34

- Graph มี vertex และ edge, ทั่วไปกว่า Tree (มี cycle ได้, ไม่จำเป็นต้องมี root)
- Adjacency List (O(V+E) space) เหมาะกับ sparse graph มากกว่า Adjacency Matrix
  (O(V²) space)
- DFS/BFS บน Graph ต้องมี `visited` set เพื่อป้องกัน infinite loop จาก cycle
  (ต่างจาก Tree traversal)
- BFS หา shortest path ใน unweighted graph ได้แม่นยำ เพราะเยี่ยมทีละระดับ
- Topological Sort จัดเรียง vertex ของ DAG ตาม dependency ใช้ Kahn's Algorithm
  (นับ in-degree + BFS)

**ต่อไป**: [Part 35 — Big O Notation, Time/Space Complexity](./part-035-big-o-notation.md)
