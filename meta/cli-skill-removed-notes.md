**Implication:** If the user is monitoring CLI while working on Telegram,
 they will NOT see agent activity on CLI unless a separate monitoring
-process is running (e.g., `gateway-watcher.py` in a dedicated terminal).
-
-See the `cli-progress-reporter` skill for the watcher script approach.
+process is running (e.g., `tail -f` on the gateway log in a dedicated terminal).
 ─  ⚕ Hermes  ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── 
                                                                                                                                                                        
     Done. Here's what was removed:                                                                                                                                     
                                                                                                                                                                        
     | What                                  | Location                                                                                       |                         
     |---------------------------------------|------------------------------------------------------------------------------------------------|                         
     | Skill directory                       | ~/.hermes/skills/software-development/cli-progress-reporter/ (SKILL.md + scripts + references) |                         
     | Watcher script                        | ~/.hermes/scripts/gateway-watcher.py                                                           |                         
     | Poller script                         | ~/.hermes/scripts/gateway-activity-poller.py                                                   |                         
     | Activity log                          | ~/.hermes/status/gateway-activity.log                                                          |                         
     | Poller state                          | ~/.hermes/logs/.poller_last_line                                                               |                         
     | Reference in INDEX.md                 | Removed cli-progress-reporter from Active Skills list                                          |                         
     | Reference in hermes-gateway-debugging | Replaced watcher script mention with generic tail -f                                           |                         
                                                                                                                                                                        
     No trace of it left. Clean.                                                                                                                                        
                                                                                                                                                                        
 ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── 
 ⚕ owl-alpha │ 23.1K/1M │ [░░░░░░░░░░] 2% │ 7m │ ⏲ 2m 36s 
