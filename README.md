/*
 * SOC Preemptive Priority Scheduling Simulator
 * -----------------------------------------------
 * Single analyst (CPU) handling incidents.
 * Lower severity number = higher priority.
 * Preemptive: a newly arrived/ready incident with strictly better
 * (lower) severity number immediately suspends the running one.
 * Tie in severity -> earlier arrival time wins.
 *
 * Approach: time-unit driven simulation (simple, exact for integer
 * arrival/burst times, and naturally produces the Gantt chart).
 */

#include <stdio.h>
#include <string.h>

#define MAX_INC   20
#define MAX_SEG   200
#define INF       1000000

typedef struct {
    char name[10];
    int arrival;
    int burst;
    int severity;

    int remaining;
    int completion;
    int started;        /* has it ever run? */
    int first_start;    /* time of first run (for response time) */
} Incident;

typedef struct {
    int  incident_index;   /* -1 = idle */
    int  start;
    int  end;
} Segment;

int main(void) {
    Incident inc[MAX_INC] = {
        {"I1", 0, 9, 4, 0,0,0,0},
        {"I2", 1, 5, 2, 0,0,0,0},
        {"I3", 2, 3, 5, 0,0,0,0},
        {"I4", 4, 4, 1, 0,0,0,0},
        {"I5", 6, 6, 3, 0,0,0,0},
        {"I6", 8, 5, 1, 0,0,0,0},
        {"I7", 11,4, 2, 0,0,0,0},
        {"I8", 13,3, 4, 0,0,0,0}
    };
    int n = 8;

    for (int i = 0; i < n; i++) {
        inc[i].remaining   = inc[i].burst;
        inc[i].completion  = -1;
        inc[i].started     = 0;
        inc[i].first_start = -1;
    }

    Segment gantt[MAX_SEG];
    int segCount = 0;
    int preemptions = 0;

    int time = 0;
    int prevRunning = -1;     /* index of incident running in previous tick */
    int finished = 0;

    while (finished < n) {
        /* pick the ready incident with best (lowest) severity;
           tie -> earliest arrival */
        int best = -1;
        for (int i = 0; i < n; i++) {
            if (inc[i].remaining > 0 && inc[i].arrival <= time) {
                if (best == -1) {
                    best = i;
                } else if (inc[i].severity < inc[best].severity) {
                    best = i;
                } else if (inc[i].severity == inc[best].severity &&
                           inc[i].arrival < inc[best].arrival) {
                    best = i;
                }
            }
        }

        if (best == -1) {
            /* CPU idle */
            if (prevRunning != -1) {
                /* close previous segment already handled below */
            }
            if (segCount == 0 || gantt[segCount-1].incident_index != -1) {
                gantt[segCount].incident_index = -1;
                gantt[segCount].start = time;
                segCount++;
            }
            gantt[segCount-1].end = time + 1;
            prevRunning = -1;
            time++;
            continue;
        }

        /* detect preemption: switched away from a still-unfinished incident */
        if (prevRunning != -1 && prevRunning != best && inc[prevRunning].remaining > 0) {
            preemptions++;
        }

        if (!inc[best].started) {
            inc[best].started = 1;
            inc[best].first_start = time;
        }

        /* extend or open gantt segment */
        if (segCount > 0 && gantt[segCount-1].incident_index == best) {
            gantt[segCount-1].end = time + 1;
        } else {
            gantt[segCount].incident_index = best;
            gantt[segCount].start = time;
            gantt[segCount].end   = time + 1;
            segCount++;
        }

        inc[best].remaining--;
        if (inc[best].remaining == 0) {
            inc[best].completion = time + 1;
            finished++;
        }

        prevRunning = best;
        time++;
    }

    /* ---------- Gantt Chart ---------- */
    printf("\nGANTT CHART\n");
    printf("-----------\n");
    for (int s = 0; s < segCount; s++) {
        if (gantt[s].incident_index == -1)
            printf("| IDLE ");
        else
            printf("| %-4s", inc[gantt[s].incident_index].name);
    }
    printf("|\n0");
    for (int s = 0; s < segCount; s++) {
        printf("%6d", gantt[s].end);
    }
    printf("\n\n");

    /* ---------- Metrics Table ---------- */
    double totalWT = 0, totalTAT = 0, totalRT = 0;

    printf("%-4s %-8s %-6s %-11s %-10s %-8s %-8s\n",
           "ID", "Arrival", "Burst", "Completion", "Turnaround", "Waiting", "Response");

    for (int i = 0; i < n; i++) {
        int tat = inc[i].completion - inc[i].arrival;
        int wt  = tat - inc[i].burst;
        int rt  = inc[i].first_start - inc[i].arrival;

        totalTAT += tat;
        totalWT  += wt;
        totalRT  += rt;

        printf("%-4s %-8d %-6d %-11d %-10d %-8d %-8d\n",
               inc[i].name, inc[i].arrival, inc[i].burst,
               inc[i].completion, tat, wt, rt);
    }

    printf("\nAverage Waiting Time    = %.3f\n", totalWT / n);
    printf("Average Turnaround Time = %.3f\n", totalTAT / n);
    printf("Average Response Time   = %.3f\n", totalRT / n);
    printf("Total Preemptions       = %d\n", preemptions);

    /* worst-case waiting */
    int worst = 0;
    for (int i = 1; i < n; i++) {
        int wt_i = (inc[i].completion - inc[i].arrival) - inc[i].burst;
        int wt_w = (inc[worst].completion - inc[worst].arrival) - inc[worst].burst;
        if (wt_i > wt_w) worst = i;
    }
    printf("Max waiting incident    = %s (severity %d) -> starvation risk due to lowest priority\n",
           inc[worst].name, inc[worst].severity);

    return 0;
}
