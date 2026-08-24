第三輪問題：勝負判定與「魂」系統

Q5. 勝負判定


https://raw.githack.com/pony0278/Blocky-Sword-Battle-Royale/86c426e44ceb38517fce99657a29eed7f1c0a57d/tools/action-studio/shield-driven-contact-coupling-lab.html


地城入侵模式的勝負條件：





入侵者獲勝：擊殺被入侵者



被入侵者獲勝：擊殺入侵者 or 撐過 3 分鐘 , 2種之中有1種成功就判斷獲勝



平手情況存在嗎？如果同時死亡,就是2者都判斷失敗,刪除當前角色存檔



Tactics5x5 對戰的勝負條件：





沿用 Tactics5x5 原本的規則嗎？沿用原本 Tactics5x5規則



有時間限制嗎？依據 Tactics5x5原本規則,獲勝者才會回到原本世界



死亡後果：





入侵者死亡：只是被踢回自己的世界？還是角色也死亡（Roguelike 永久死亡）？永久死亡



被入侵者死亡：角色永久死亡嗎？還是有特殊保護？永久死亡

Q6.「魂」系統詳細設計





魂的獲取：





Boss 掉落多少魂？1



寶箱掉落多少魂？1



還有其他獲取方式嗎？花錢購買(未來會有付費機制)



魂的消耗：





發起一次入侵需要消耗多少魂？1



魂還有其他用途嗎？（例如：強化裝備？購買物品？）無



魂的掠奪：





入侵獲勝可以搶走對方多少魂？對方身上最大魂數-1(例如:總共5,5-(5-1))



被入侵者獲勝可以獲得入侵者的魂嗎？可以,如果入侵者身上還有魂,就搶走"對方身上最大魂數-1"



強制離線時雙方各扣多少魂？對方身上最大魂數-1

Q7. 復仇機制





復仇觸發：





被入侵並輸掉後，會收到「復仇機會」通知嗎？會



復仇有時間限制嗎？48小時



復仇流程：





復仇是直接入侵對方？還是強制 Tactics5x5 對戰？



復仇需要消耗魂嗎？



復仇獎勵：





「掠奪對方全部積分」— 這是指 PvP 積分？還是魂？還是兩者？



如果對方積分是 0，復仇成功有其他獎勵嗎？


● Based on the current project state and the roadmap in CLAUDE.md, here are my suggestions:

  Next Steps Options

  Option 1: Continue V3 Roadmap (Recommended)

  Per /docs/V3_IMPLEMENTATION_ROADMAP.md, Phase 4 is Hero Skill System:

  | Priority | Task                                 | Complexity |
  |----------|--------------------------------------|------------|
  | High     | Implement basic skill infrastructure | Medium     |
  | High     | Add skill cooldown tracking          | Low        |
  | High     | Implement 2-3 core skills first      | Medium     |

  This is the natural next step but involves significant work (201 test cases in SKILL_SYSTEM_V3_TESTPLAN.md).

  ---
  Option 2: Complete Pending V3 Mechanics

  There are simpler V3 mechanics not yet implemented:

  | Task                                       | Complexity | Benefit                    |
  |--------------------------------------------|------------|----------------------------|
  | Minion Decay (-1 HP/round)                 | Low        | Core game balance          |
  | Round 8 Pressure (-1 HP/round to all)      | Low        | Late-game urgency          |
  | Death Choice (spawn obstacle or BUFF tile) | Medium     | Complete minion death flow |

  These are smaller, self-contained features that could be done before the larger Skill System.

  ---
  Option 3: Refactoring Checkpoint

  RuleEngine.java is now ~2000 lines. A refactoring pass could:

  | Task                                              | Benefit                       |
  |---------------------------------------------------|-------------------------------|
  | Extract TurnManager class                         | Cleaner turn/round handling   |
  | Extract CombatResolver class                      | Isolate damage/Guardian logic |
  | Replace remaining new Unit(...) with unit.with*() | Reduce boilerplate            |

  This would improve maintainability before adding more features.

  ---
  Option 4: Write Integration Tests

  Current tests are mostly unit tests. Could add:
  - Full game flow tests (start → play → victory)
  - Multi-round scenario tests
  - Client-server integration tests

  ---
  My Recommendation

  Option 2: Complete Pending V3 Mechanics (Minion Decay + Round 8 Pressure)

  Reasons:
  1. Low complexity, quick wins
  2. These are required for proper game balance
  3. Build confidence before larger Skill System
  4. Round-end processing infrastructure already exists

  Estimated scope: ~30 minutes, ~10-15 new tests

  Would you like me to proceed with Option 2, or do you prefer a different direction?

