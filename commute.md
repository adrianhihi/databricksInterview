If I treat it as a general weighted shortest-path problem, I’d use Dijkstra with a priority queue, which gives me O(kMN\log(kMN)).
But here, once a mode is chosen, all steps have the same time and cost for that mode, so the shortest-time path for that mode is equivalent to the fewest-block path.
That allows me to do a single BFS over (row, col, mode) in O(kMN), which is asymptotically better

```java
import java.util.*;

public class Solution {

    // State in BFS: position + chosen mode + how many blocks of this mode we used
    static class State {
        int r, c;
        int mode;    // index in modes[]
        int blocks;  // number of road blocks for this mode

        State(int r, int c, int mode, int blocks) {
            this.r = r;
            this.c = c;
            this.mode = mode;
            this.blocks = blocks;
        }
    }

    // 4-directional moves
    private static final int[][] DIRS = {
        {1, 0}, {-1, 0}, {0, 1}, {0, -1}
    };

    public String findOptimalCommute(char[][] grid, String[] modes, int[] costs, int[] times) {
        if (grid == null || grid.length == 0 || grid[0].length == 0) {
            return "";
        }

        int rows = grid.length;
        int cols = grid[0].length;

        int sr = -1, sc = -1;
        int dr = -1, dc = -1;

        // 1) Locate S and D
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == 'S') {
                    sr = r; sc = c;
                } else if (grid[r][c] == 'D') {
                    dr = r; dc = c;
                }
            }
        }

        if (sr == -1 || dr == -1) return "";

        // Edge case: S == D (not expected by problem, but safe-guard)
        if (sr == dr && sc == dc) {
            return modes.length > 0 ? modes[0] : "";
        }

        // 2) Check if S is directly adjacent to D (0 blocks, 0 time, 0 cost)
        for (int[] d : DIRS) {
            int nr = sr + d[0];
            int nc = sc + d[1];
            if (inBounds(nr, nc, rows, cols) && grid[nr][nc] == 'D') {
                // Any mode is fine; choose modes[0] if exists
                return modes.length > 0 ? modes[0] : "";
            }
        }

        // 3) BFS over (r, c, mode). visited[mode][r][c] avoids revisiting same state.
        boolean[][][] visited = new boolean[modes.length][rows][cols];
        Queue<State> queue = new ArrayDeque<>();

        // We will track the minimal number of blocks for each mode that can reach D.
        int[] bestBlocks = new int[modes.length];
        Arrays.fill(bestBlocks, Integer.MAX_VALUE);

        // Initialize queue from neighbors of S: this is where we "choose" a mode
        for (int[] d : DIRS) {
            int nr = sr + d[0];
            int nc = sc + d[1];
            if (!inBounds(nr, nc, rows, cols)) continue;
            char ch = grid[nr][nc];

            if (ch >= '1' && ch <= '9') {
                int modeIndex = ch - '1';
                if (modeIndex >= 0 && modeIndex < modes.length) {
                    if (!visited[modeIndex][nr][nc]) {
                        visited[modeIndex][nr][nc] = true;
                        // stepping onto the first road block of this mode
                        queue.offer(new State(nr, nc, modeIndex, 1));
                    }
                }
            }
            // 'X' or 'S' or 'D' are skipped here; D was handled in the adjacency check above.
        }

        // 4) BFS
        while (!queue.isEmpty()) {
            State cur = queue.poll();
            int r = cur.r;
            int c = cur.c;
            int m = cur.mode;
            int blocks = cur.blocks;

            // If we reached D with this mode, update bestBlocks for this mode.
            if (r == dr && c == dc) {
                if (blocks < bestBlocks[m]) {
                    bestBlocks[m] = blocks;
                }
                // Do NOT expand further from D
                continue;
            }

            // Expand neighbors with the same mode or D
            for (int[] d : DIRS) {
                int nr = r + d[0];
                int nc = c + d[1];
                if (!inBounds(nr, nc, rows, cols)) continue;
                char ch = grid[nr][nc];
                if (ch == 'X') continue; // cannot pass roadblocks

                if (ch == 'D') {
                    // We can reach D without increasing block count
                    if (!visited[m][nr][nc]) {
                        visited[m][nr][nc] = true;
                        queue.offer(new State(nr, nc, m, blocks));
                    }
                } else if (ch == (char)('1' + m)) {
                    // Continue on the same mode's road
                    if (!visited[m][nr][nc]) {
                        visited[m][nr][nc] = true;
                        queue.offer(new State(nr, nc, m, blocks + 1));
                    }
                }
                // Different digits => cannot switch mode, so skip.
            }
        }

        // 5) After BFS, compute best (time, cost) among all modes that can reach D
        long bestTime = Long.MAX_VALUE;
        long bestCost = Long.MAX_VALUE;
        int bestMode = -1;

        for (int i = 0; i < modes.length; i++) {
            if (bestBlocks[i] == Integer.MAX_VALUE) continue; // this mode cannot reach D

            long totalTime = (long) bestBlocks[i] * times[i];
            long totalCost = (long) bestBlocks[i] * costs[i];

            if (totalTime < bestTime || (totalTime == bestTime && totalCost < bestCost)) {
                bestTime = totalTime;
                bestCost = totalCost;
                bestMode = i;
            }
        }

        return bestMode == -1 ? "" : modes[bestMode];
    }

    private boolean inBounds(int r, int c, int rows, int cols) {
        return r >= 0 && r < rows && c >= 0 && c < cols;
    }
}
```
