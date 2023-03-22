# TPC-H Query Analysis

In this doc, I will analyze characters of TPC-H queries. These characters includes table number, join number, join type and whether we could inject true cardinality for each query in postgres and why if we could not.

Something undetermined is that whether not only the join relations but also the operation including **group by**, **order by**, **subplan** make the TPC-H difficult ? 

| Query | Table Number | Join Number |   Join Type    | Could inject true cardinality in postgres |                     Why couldn't reject                      |
| :---: | :----------: | :---------: | :------------: | :---------------------------------------: | :----------------------------------------------------------: |
|   1   |      1       |      0      |                |                  **Yes**                  |                                                              |
|   2   |      5       |      9      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|   3   |      3       |      2      |     inner      |                  **Yes**                  |                                                              |
|   4   |      2       |      1      |     inner      |                  **Yes**                  |                                                              |
|   5   |      6       |      6      |     inner      |                  **Yes**                  |                                                              |
|   6   |      1       |      0      |                |                  **Yes**                  |                                                              |
|   7   |      6       |      5      |     inner      |      **Yes (need to modify query)**       |                                                              |
|   8   |      8       |      7      |     inner      |      **Yes (need to modify query)**       |                                                              |
|   9   |      6       |      6      |     inner      |      **Yes (need to modify query)**       |                                                              |
|  10   |      4       |      3      |     inner      |                  **Yes**                  |                                                              |
|  11   |      3       |      4      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|  12   |      2       |      2      |     inner      |                  **Yes**                  |                                                              |
|  13   |      2       |      1      | **left outer** |                  **Yes**                  |                                                              |
|  14   |      2       |      1      |     inner      |                  **Yes**                  |                                                              |
|  15   |      2       |      1      |     inner      |                    No                     |  can not distinguish the tables in both main plan and view   |
|  16   |      3       |      2      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|  17   |      3       |      2      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|  18   |      4       |      3      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|  19   |      2       |      1      |     inner      |                  **Yes**                  |                                                              |
|  20   |      3       |      2      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|  21   |      6       |      5      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |
|  22   |      2       |      1      |     inner      |                    No                     | can not distinguish the tables in both main plan and subplan |

  