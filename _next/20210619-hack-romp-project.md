# Hack ROMP -- aggiornare codice async legacy

## Note
* Analisi su e7cd7ab
* 3 check warnings
* Un numero indeterminato di clippy lints
* I test passano, ci sono warnings
* Risultati di coverage per cargo tarpaulin:
  ```
  || Tested/Total Lines:
  || src/body_parser/fixed_length.rs: 52/80
  || src/body_parser/text.rs: 43/53
  || src/bootstrap.rs: 0/77
  || src/config/config.rs: 29/78
  || src/config/log.rs: 7/10
  || src/config/user.rs: 40/51
  || src/downstream.rs: 0/68
  || src/init.rs: 12/27
  || src/main.rs: 0/2
  || src/message/request.rs: 0/25
  || src/message/response.rs: 0/104
  || src/message/serializer.rs: 207/236
  || src/message/stomp_message.rs: 232/278
  || src/parser.rs: 209/299
  || src/persist/mlog.rs: 23/39
  || src/session/ecg.rs: 65/81
  || src/session/filter.rs: 65/89
  || src/session/interruptible_interval.rs: 0/20
  || src/session/mq.rs: 7/65
  || src/session/reader.rs: 0/229
  || src/session/stomp_session.rs: 56/250
  || src/session/subscription.rs: 0/62
  || src/session/writer.rs: 0/96
  || src/system/limits.rs: 2/2
  || src/util.rs: 13/15
  || src/web_socket/ws_demunge.rs: 309/357
  || src/web_socket/ws_headers.rs: 102/129
  || src/web_socket/ws_ports.rs: 0/4
  || src/web_socket/ws_response.rs: 39/44
  || src/workflow/admin_router.rs: 0/13
  || src/workflow/console/filter.rs: 1/21
  || src/workflow/console/ngin.rs: 106/229
  || src/workflow/context.rs: 0/28
  || src/workflow/destination/destination.rs: 128/285
  || src/workflow/destination/server.rs: 64/150
  || src/workflow/filter/ack.rs: 0/15
  || src/workflow/filter/admin/clean.rs: 0/22
  || src/workflow/filter/admin/perms.rs: 0/63
  || src/workflow/filter/admin/shutdown.rs: 0/13
  || src/workflow/filter/admin.rs: 0/17
  || src/workflow/filter/closure.rs: 0/6
  || src/workflow/filter/connect.rs: 0/80
  || src/workflow/filter/disconnect.rs: 0/6
  || src/workflow/filter/echo.rs: 0/30
  || src/workflow/filter/http/closure.rs: 0/6
  || src/workflow/filter/http/get.rs: 0/14
  || src/workflow/filter/http/health_check.rs: 0/10
  || src/workflow/filter/http/hello.rs: 0/10
  || src/workflow/filter/http/not_found.rs: 0/10
  || src/workflow/filter/http/upgrade.rs: 0/38
  || src/workflow/filter/http/version.rs: 0/9
  || src/workflow/filter/http.rs: 0/14
  || src/workflow/filter/log.rs: 0/28
  || src/workflow/filter/persist.rs: 0/21
  || src/workflow/filter/ping.rs: 0/10
  || src/workflow/filter/receipt.rs: 0/19
  || src/workflow/filter/requires_auth.rs: 0/56
  || src/workflow/filter/resend.rs: 0/42
  || src/workflow/filter/send.rs: 0/50
  || src/workflow/filter/subscribe.rs: 0/89
  || src/workflow/http_router.rs: 3/38
  || src/workflow/internal.rs: 0/23
  || src/workflow/router.rs: 0/53
  || src/workflow/sha_auth.rs: 31/33
  ||
  41.73% coverage, 1845/4421 lines covered
  ```
* cargo audit:
  ```
  Crate:         arc-swap
  Version:       0.4.4
  Title:         Dangling reference in `access::Map` with Constant
  Date:          2020-12-10
  ID:            RUSTSEC-2020-0091
  URL:           https://rustsec.org/advisories/RUSTSEC-2020-0091
  Solution:      Upgrade to >=0.4.8, <1.0.0-0 OR >=1.1.0
  Dependency tree:
  arc-swap 0.4.4
  ├── signal-hook-registry 1.2.0
  │   └── signal-hook 0.1.13
  │       └── tokio-signal 0.2.7
  │           └── romp 0.5.2
  └── log4rs 1.0.0
      └── romp 0.5.2
  
  Crate:         smallvec
  Version:       0.6.13
  Title:         Buffer overflow in SmallVec::insert_many
  Date:          2021-01-08
  ID:            RUSTSEC-2021-0003
  URL:           https://rustsec.org/advisories/RUSTSEC-2021-0003
  Solution:      Upgrade to >=0.6.14, <1.0.0 OR >=1.6.1
  Dependency tree:
  smallvec 0.6.13
  
  Crate:         net2
  Version:       0.2.33
  Warning:       unmaintained
  Title:         `net2` crate has been deprecated; use `socket2` instead
  Date:          2020-05-01
  ID:            RUSTSEC-2020-0016
  URL:           https://rustsec.org/advisories/RUSTSEC-2020-0016
  Dependency tree:
  net2 0.2.33
  ├── miow 0.2.1
  │   └── mio 0.6.21
  │       ├── tokio-uds 0.2.5
  │       │   └── tokio 0.1.22
  │       │       └── romp 0.5.2
  │       ├── tokio-udp 0.1.5
  │       │   └── tokio 0.1.22
  │       ├── tokio-tcp 0.1.3
  │       │   └── tokio 0.1.22
  │       ├── tokio-signal 0.2.7
  │       │   └── romp 0.5.2
  │       ├── tokio-reactor 0.1.11
  │       │   ├── tokio-uds 0.2.5
  │       │   ├── tokio-udp 0.1.5
  │       │   ├── tokio-tcp 0.1.3
  │       │   ├── tokio-signal 0.2.7
  │       │   └── tokio 0.1.22
  │       ├── tokio 0.1.22
  │       └── mio-uds 0.6.7
  │           ├── tokio-uds 0.2.5
  │           └── tokio-signal 0.2.7
  └── mio 0.6.21
  
  Crate:         arc-swap
  Version:       0.4.4
  Warning:       yanked
  
  Crate:         miow
  Version:       0.2.1
  Warning:       yanked
  Dependency tree:
  miow 0.2.1
  └── mio 0.6.21
      ├── tokio-uds 0.2.5
      │   └── tokio 0.1.22
      │       └── romp 0.5.2
      ├── tokio-udp 0.1.5
      │   └── tokio 0.1.22
      ├── tokio-tcp 0.1.3
      │   └── tokio 0.1.22
      ├── tokio-signal 0.2.7
      │   └── romp 0.5.2
      ├── tokio-reactor 0.1.11
      │   ├── tokio-uds 0.2.5
      │   ├── tokio-udp 0.1.5
      │   ├── tokio-tcp 0.1.3
      │   ├── tokio-signal 0.2.7
      │   └── tokio 0.1.22
      ├── tokio 0.1.22
      └── mio-uds 0.6.7
          ├── tokio-uds 0.2.5
          └── tokio-signal 0.2.7
  
  Crate:         net2
  Version:       0.2.33
  Warning:       yanked
  
  Crate:         quote
  Version:       1.0.2
  Warning:       yanked
  Dependency tree:
  quote 1.0.2
  ├── thiserror-impl 1.0.25
  │   └── thiserror 1.0.25
  │       └── log4rs 1.0.0
  │           └── romp 0.5.2
  ├── syn 1.0.73
  │   ├── thiserror-impl 1.0.25
  │   ├── serde_derive 1.0.104
  │   │   ├── serde 1.0.104
  │   │   │   ├── toml 0.5.8
  │   │   │   │   └── romp 0.5.2
  │   │   │   ├── serde_yaml 0.8.11
  │   │   │   │   └── log4rs 1.0.0
  │   │   │   ├── serde_json 1.0.45
  │   │   │   │   └── log4rs 1.0.0
  │   │   │   ├── serde-value 0.7.0
  │   │   │   │   └── log4rs 1.0.0
  │   │   │   ├── romp 0.5.2
  │   │   │   ├── log4rs 1.0.0
  │   │   │   └── log 0.4.8
  │   │   │       ├── tokio-uds 0.2.5
  │   │   │       │   └── tokio 0.1.22
  │   │   │       │       └── romp 0.5.2
  │   │   │       ├── tokio-udp 0.1.5
  │   │   │       │   └── tokio 0.1.22
  │   │   │       ├── tokio-threadpool 0.1.17
  │   │   │       │   ├── tokio-fs 0.1.6
  │   │   │       │   │   └── tokio 0.1.22
  │   │   │       │   └── tokio 0.1.22
  │   │   │       ├── tokio-reactor 0.1.11
  │   │   │       │   ├── tokio-uds 0.2.5
  │   │   │       │   ├── tokio-udp 0.1.5
  │   │   │       │   ├── tokio-tcp 0.1.3
  │   │   │       │   │   └── tokio 0.1.22
  │   │   │       │   ├── tokio-signal 0.2.7
  │   │   │       │   │   └── romp 0.5.2
  │   │   │       │   └── tokio 0.1.22
  │   │   │       ├── tokio-io 0.1.12
  │   │   │       │   ├── tokio-uds 0.2.5
  │   │   │       │   ├── tokio-udp 0.1.5
  │   │   │       │   ├── tokio-tcp 0.1.3
  │   │   │       │   ├── tokio-signal 0.2.7
  │   │   │       │   ├── tokio-reactor 0.1.11
  │   │   │       │   ├── tokio-fs 0.1.6
  │   │   │       │   ├── tokio-codec 0.1.1
  │   │   │       │   │   ├── tokio-uds 0.2.5
  │   │   │       │   │   ├── tokio-udp 0.1.5
  │   │   │       │   │   └── tokio 0.1.22
  │   │   │       │   └── tokio 0.1.22
  │   │   │       ├── romp 0.5.2
  │   │   │       ├── mio 0.6.21
  │   │   │       │   ├── tokio-uds 0.2.5
  │   │   │       │   ├── tokio-udp 0.1.5
  │   │   │       │   ├── tokio-tcp 0.1.3
  │   │   │       │   ├── tokio-signal 0.2.7
  │   │   │       │   ├── tokio-reactor 0.1.11
  │   │   │       │   ├── tokio 0.1.22
  │   │   │       │   └── mio-uds 0.6.7
  │   │   │       │       ├── tokio-uds 0.2.5
  │   │   │       │       └── tokio-signal 0.2.7
  │   │   │       └── log4rs 1.0.0
  │   │   └── romp 0.5.2
  │   └── derivative 2.2.0
  │       └── log4rs 1.0.0
  ├── serde_derive 1.0.104
  └── derivative 2.2.0
  
  error: 2 vulnerabilities found!
  warning: 5 allowed warnings found
  ```
* cargo updated:
  ```
  Name                                  Project  Compat   Latest   Kind         Platform
  ----                                  -------  ------   ------   ----         --------
  bytes->byteorder                      1.3.2    1.4.3    1.4.3    Normal       ---
  bytes->byteorder                      1.3.2    1.4.3    Removed  Normal       ---
  bytes->iovec                          0.1.4    ---      Removed  Normal       ---
  chrono->libc                          0.2.96   0.2.97   0.2.97   Normal       ---
  chrono->num-integer                   0.1.42   0.1.44   0.1.44   Normal       ---
  chrono->num-traits                    0.2.11   0.2.14   0.2.14   Normal       ---
  chrono->winapi                        0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  cloudabi->bitflags                    1.2.1    ---      Removed  Normal       ---
  cpufeatures->libc                     0.2.96   0.2.97   0.2.97   Normal       aarch64-apple-darwin
  crossbeam-deque->crossbeam-epoch      0.8.0    0.8.2    Removed  Normal       ---
  crossbeam-deque->crossbeam-utils      0.7.0    0.7.2    Removed  Normal       ---
  crossbeam-epoch->autocfg              0.1.7    1.0.1    Removed  Build        ---
  crossbeam-epoch->cfg-if               0.1.10   ---      Removed  Normal       ---
  crossbeam-epoch->crossbeam-utils      0.7.0    0.7.2    Removed  Normal       ---
  crossbeam-epoch->lazy_static          1.4.0    ---      Removed  Normal       ---
  crossbeam-epoch->memoffset            0.5.3    0.5.6    Removed  Normal       ---
  crossbeam-epoch->scopeguard           1.1.0    ---      Removed  Normal       ---
  crossbeam-queue->crossbeam-utils      0.6.6    0.7.2    Removed  Normal       ---
  crossbeam-utils->autocfg              0.1.7    1.0.1    Removed  Build        ---
  crossbeam-utils->cfg-if               0.1.10   ---      Removed  Normal       ---
  crossbeam-utils->lazy_static          1.4.0    ---      Removed  Normal       ---
  derivative->quote                     1.0.2    1.0.9    1.0.9    Normal       ---
  fuchsia-zircon->bitflags              1.2.1    ---      Removed  Normal       ---
  fuchsia-zircon->fuchsia-zircon-sys    0.3.3    ---      Removed  Normal       ---
  futures                               0.1.29   0.1.31   0.3.15   Normal       ---
  hermit-abi->libc                      0.2.96   0.2.97   0.2.97   Normal       ---
  hermit-abi->libc                      0.2.96   0.2.97   Removed  Normal       ---
  iovec->libc                           0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  iovec->libc                           0.2.96   0.2.97   Removed  Normal       cfg(unix)
  kernel32-sys->winapi                  0.2.8    ---      Removed  Normal       ---
  kernel32-sys->winapi-build            0.1.1    ---      Removed  Build        ---
  libc                                  0.2.96   0.2.97   0.2.97   Normal       ---
  lock_api->scopeguard                  1.1.0    ---      Removed  Normal       ---
  log                                   0.4.8    0.4.14   0.4.14   Normal       ---
  log->cfg-if                           0.1.10   1.0.0    1.0.0    Normal       ---
  log->cfg-if                           0.1.10   1.0.0    Removed  Normal       ---
  log->serde                            1.0.104  1.0.126  1.0.126  Normal       ---
  log->serde                            1.0.104  1.0.126  Removed  Normal       ---
  log4rs->arc-swap                      0.4.4    0.4.8    0.4.8    Normal       ---
  log4rs->fnv                           1.0.6    1.0.7    1.0.7    Normal       ---
  log4rs->libc                          0.2.96   0.2.97   0.2.97   Normal       cfg(not(windows))
  log4rs->log                           0.4.8    0.4.14   0.4.14   Normal       ---
  log4rs->serde                         1.0.104  1.0.126  1.0.126  Normal       ---
  log4rs->serde_json                    1.0.45   1.0.64   1.0.64   Normal       ---
  log4rs->serde_yaml                    0.8.11   0.8.17   0.8.17   Normal       ---
  log4rs->winapi                        0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  memoffset->rustc_version              0.2.3    Removed  Removed  Build        ---
  mio->cfg-if                           0.1.10   ---      Removed  Normal       ---
  mio->fuchsia-zircon                   0.3.3    ---      Removed  Normal       cfg(target_os = "fuchsia")
  mio->fuchsia-zircon-sys               0.3.3    ---      Removed  Normal       cfg(target_os = "fuchsia")
  mio->iovec                            0.1.4    ---      Removed  Normal       ---
  mio->kernel32-sys                     0.2.2    ---      Removed  Normal       cfg(windows)
  mio->libc                             0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  mio->libc                             0.2.96   0.2.97   Removed  Normal       cfg(unix)
  mio->log                              0.4.8    0.4.14   0.4.14   Normal       ---
  mio->log                              0.4.8    0.4.14   Removed  Normal       ---
  mio->miow                             0.2.1    0.2.2    0.2.2    Normal       cfg(windows)
  mio->miow                             0.2.1    0.2.2    Removed  Normal       cfg(windows)
  mio->net2                             0.2.33   0.2.37   0.2.37   Normal       ---
  mio->net2                             0.2.33   0.2.37   Removed  Normal       ---
  mio->slab                             0.4.2    0.4.3    0.4.3    Normal       ---
  mio->slab                             0.4.2    0.4.3    Removed  Normal       ---
  mio->winapi                           0.2.8    ---      Removed  Normal       cfg(windows)
  mio-uds->iovec                        0.1.4    ---      Removed  Normal       cfg(unix)
  mio-uds->libc                         0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  mio-uds->libc                         0.2.96   0.2.97   Removed  Normal       cfg(unix)
  mio-uds->mio                          0.6.21   0.6.23   0.6.23   Normal       cfg(unix)
  mio-uds->mio                          0.6.21   0.6.23   Removed  Normal       cfg(unix)
  miow->kernel32-sys                    0.2.2    ---      Removed  Normal       ---
  miow->net2                            0.2.33   0.2.37   0.2.37   Normal       ---
  miow->net2                            0.2.33   0.2.37   Removed  Normal       ---
  miow->winapi                          0.2.8    ---      Removed  Normal       ---
  miow->ws2_32-sys                      0.2.1    ---      Removed  Normal       ---
  net2->cfg-if                          0.1.10   ---      Removed  Normal       ---
  net2->libc                            0.2.96   0.2.97   0.2.97   Normal       cfg(any(target_os = "redox", unix))
  net2->libc                            0.2.96   0.2.97   Removed  Normal       cfg(any(target_os = "redox", unix))
  net2->winapi                          0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  net2->winapi                          0.3.8    0.3.9    Removed  Normal       cfg(windows)
  num-integer->autocfg                  1.0.0    1.0.1    1.0.1    Build        ---
  num-integer->num-traits               0.2.11   0.2.14   0.2.14   Normal       ---
  num-traits->autocfg                   1.0.0    1.0.1    1.0.1    Build        ---
  num_cpus->hermit-abi                  0.1.6    0.1.18   0.1.18   Normal       cfg(all(any(target_arch = "x86_64", target_arch = "aarch64"), target_os = "hermit"))
  num_cpus->hermit-abi                  0.1.6    0.1.18   Removed  Normal       cfg(all(any(target_arch = "x86_64", target_arch = "aarch64"), target_os = "hermit"))
  num_cpus->libc                        0.2.96   0.2.97   0.2.97   Normal       ---
  num_cpus->libc                        0.2.96   0.2.97   Removed  Normal       ---
  ordered-float->num-traits             0.2.11   0.2.14   0.2.14   Normal       ---
  owning_ref->stable_deref_trait        1.1.1    1.2.0    1.2.0    Normal       ---
  parking_lot->lock_api                 0.3.3    0.3.4    0.3.4    Normal       ---
  parking_lot->lock_api                 0.3.3    0.3.4    Removed  Normal       ---
  parking_lot->parking_lot_core         0.6.2    ---      Removed  Normal       ---
  parking_lot->rustc_version            0.2.3    ---      Removed  Build        ---
  parking_lot_core->cfg-if              0.1.10   ---      Removed  Normal       ---
  parking_lot_core->cloudabi            0.0.3    ---      Removed  Normal       cfg(target_os = "cloudabi")
  parking_lot_core->libc                0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  parking_lot_core->libc                0.2.96   0.2.97   Removed  Normal       cfg(unix)
  parking_lot_core->redox_syscall       0.1.56   0.1.57   0.1.57   Normal       cfg(target_os = "redox")
  parking_lot_core->redox_syscall       0.1.56   0.1.57   Removed  Normal       cfg(target_os = "redox")
  parking_lot_core->redox_syscall       0.2.8    0.2.9    0.2.9    Normal       cfg(target_os = "redox")
  parking_lot_core->rustc_version       0.2.3    ---      Removed  Build        ---
  parking_lot_core->smallvec            0.6.13   0.6.14   0.6.14   Normal       ---
  parking_lot_core->smallvec            0.6.13   0.6.14   Removed  Normal       ---
  parking_lot_core->winapi              0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  parking_lot_core->winapi              0.3.8    0.3.9    Removed  Normal       cfg(windows)
  proc-macro2->unicode-xid              0.2.0    0.2.2    0.2.2    Normal       ---
  proc-macro2->unicode-xid              0.2.0    0.2.2    Removed  Normal       ---
  quote->proc-macro2                    1.0.27   ---      Removed  Normal       ---
  rand->libc                            0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  rand->winapi                          0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  rustc_version->semver                 0.9.0    ---      Removed  Normal       ---
  rustc_version->semver                 0.9.0    Removed  Removed  Normal       ---
  semver->semver-parser                 0.7.0    ---      Removed  Normal       ---
  semver->semver-parser                 0.7.0    Removed  Removed  Normal       ---
  serde                                 1.0.104  1.0.126  1.0.126  Normal       ---
  serde->serde_derive                   1.0.104  1.0.126  1.0.126  Normal       ---
  serde->serde_derive                   1.0.104  1.0.126  Removed  Normal       ---
  serde-value->serde                    1.0.104  1.0.126  1.0.126  Normal       ---
  serde_derive                          1.0.104  1.0.126  1.0.126  Normal       ---
  serde_derive->proc-macro2             1.0.27   ---      Removed  Normal       ---
  serde_derive->quote                   1.0.2    1.0.9    1.0.9    Normal       ---
  serde_derive->quote                   1.0.2    1.0.9    Removed  Normal       ---
  serde_derive->syn                     1.0.73   ---      Removed  Normal       ---
  serde_json->itoa                      0.4.5    0.4.7    0.4.7    Normal       ---
  serde_json->ryu                       1.0.2    1.0.5    1.0.5    Normal       ---
  serde_json->serde                     1.0.104  1.0.126  1.0.126  Normal       ---
  serde_yaml->dtoa                      0.4.4    0.4.8    0.4.8    Normal       ---
  serde_yaml->linked-hash-map           0.5.2    0.5.4    0.5.4    Normal       ---
  serde_yaml->serde                     1.0.104  1.0.126  1.0.126  Normal       ---
  serde_yaml->yaml-rust                 0.4.3    0.4.5    0.4.5    Normal       ---
  signal-hook->libc                     0.2.96   Removed  Removed  Normal       ---
  signal-hook->signal-hook-registry     1.2.0    Removed  Removed  Normal       ---
  signal-hook-registry->arc-swap        0.4.4    Removed  Removed  Normal       ---
  signal-hook-registry->libc            0.2.96   Removed  Removed  Normal       ---
  smallvec->maybe-uninit                2.0.0    ---      Removed  Normal       ---
  syn->proc-macro2                      1.0.27   ---      Removed  Normal       ---
  syn->quote                            1.0.2    1.0.9    1.0.9    Normal       ---
  syn->quote                            1.0.2    1.0.9    Removed  Normal       ---
  syn->unicode-xid                      0.2.0    0.2.2    0.2.2    Normal       ---
  syn->unicode-xid                      0.2.0    0.2.2    Removed  Normal       ---
  thiserror-impl->quote                 1.0.2    1.0.9    1.0.9    Normal       ---
  thread-id->libc                       0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  thread-id->redox_syscall              0.1.56   0.1.57   0.1.57   Normal       cfg(target_os = "redox")
  thread-id->winapi                     0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  time->libc                            0.2.96   0.2.97   0.2.97   Normal       ---
  time->winapi                          0.3.8    0.3.9    0.3.9    Development  ---
  tokio                                 0.1.22   ---      1.7.1    Normal       ---
  tokio->bytes                          0.4.12   ---      Removed  Normal       ---
  tokio->futures                        0.1.29   0.1.31   Removed  Normal       ---
  tokio->mio                            0.6.21   0.6.23   Removed  Normal       ---
  tokio->num_cpus                       1.12.0   1.13.0   Removed  Normal       ---
  tokio->tokio-codec                    0.1.1    0.1.2    Removed  Normal       ---
  tokio->tokio-current-thread           0.1.6    0.1.7    Removed  Normal       ---
  tokio->tokio-executor                 0.1.9    0.1.10   Removed  Normal       ---
  tokio->tokio-fs                       0.1.6    0.1.7    Removed  Normal       ---
  tokio->tokio-io                       0.1.12   0.1.13   Removed  Normal       ---
  tokio->tokio-reactor                  0.1.11   0.1.12   Removed  Normal       ---
  tokio->tokio-sync                     0.1.7    0.1.8    Removed  Normal       ---
  tokio->tokio-tcp                      0.1.3    0.1.4    Removed  Normal       ---
  tokio->tokio-threadpool               0.1.17   0.1.18   Removed  Normal       ---
  tokio->tokio-timer                    0.2.12   0.2.13   Removed  Normal       ---
  tokio->tokio-udp                      0.1.5    0.1.6    Removed  Normal       ---
  tokio->tokio-uds                      0.2.5    0.2.7    Removed  Normal       cfg(unix)
  tokio-codec->bytes                    0.4.12   ---      Removed  Normal       ---
  tokio-codec->futures                  0.1.29   0.1.31   Removed  Normal       ---
  tokio-codec->tokio-io                 0.1.12   0.1.13   Removed  Normal       ---
  tokio-current-thread->futures         0.1.29   0.1.31   Removed  Normal       ---
  tokio-current-thread->tokio-executor  0.1.9    0.1.10   Removed  Normal       ---
  tokio-executor->crossbeam-utils       0.6.6    0.7.2    0.7.2    Normal       ---
  tokio-executor->crossbeam-utils       0.6.6    0.7.2    Removed  Normal       ---
  tokio-executor->futures               0.1.29   0.1.31   0.1.31   Normal       ---
  tokio-executor->futures               0.1.29   0.1.31   Removed  Normal       ---
  tokio-fs->futures                     0.1.29   0.1.31   Removed  Normal       ---
  tokio-fs->tokio-io                    0.1.12   0.1.13   Removed  Normal       ---
  tokio-fs->tokio-threadpool            0.1.17   0.1.18   Removed  Normal       ---
  tokio-io->bytes                       0.4.12   ---      Removed  Normal       ---
  tokio-io->futures                     0.1.29   0.1.31   0.1.31   Normal       ---
  tokio-io->futures                     0.1.29   0.1.31   Removed  Normal       ---
  tokio-io->log                         0.4.8    0.4.14   0.4.14   Normal       ---
  tokio-io->log                         0.4.8    0.4.14   Removed  Normal       ---
  tokio-reactor->crossbeam-utils        0.6.6    0.7.2    0.7.2    Normal       ---
  tokio-reactor->crossbeam-utils        0.6.6    0.7.2    Removed  Normal       ---
  tokio-reactor->futures                0.1.29   0.1.31   0.1.31   Normal       ---
  tokio-reactor->futures                0.1.29   0.1.31   Removed  Normal       ---
  tokio-reactor->lazy_static            1.4.0    ---      Removed  Normal       ---
  tokio-reactor->log                    0.4.8    0.4.14   0.4.14   Normal       ---
  tokio-reactor->log                    0.4.8    0.4.14   Removed  Normal       ---
  tokio-reactor->mio                    0.6.21   0.6.23   0.6.23   Normal       ---
  tokio-reactor->mio                    0.6.21   0.6.23   Removed  Normal       ---
  tokio-reactor->num_cpus               1.12.0   1.13.0   1.13.0   Normal       ---
  tokio-reactor->num_cpus               1.12.0   1.13.0   Removed  Normal       ---
  tokio-reactor->parking_lot            0.9.0    ---      Removed  Normal       ---
  tokio-reactor->slab                   0.4.2    0.4.3    0.4.3    Normal       ---
  tokio-reactor->slab                   0.4.2    0.4.3    Removed  Normal       ---
  tokio-reactor->tokio-executor         0.1.9    0.1.10   0.1.10   Normal       ---
  tokio-reactor->tokio-executor         0.1.9    0.1.10   Removed  Normal       ---
  tokio-reactor->tokio-io               0.1.12   0.1.13   0.1.13   Normal       ---
  tokio-reactor->tokio-io               0.1.12   0.1.13   Removed  Normal       ---
  tokio-reactor->tokio-sync             0.1.7    0.1.8    0.1.8    Normal       ---
  tokio-reactor->tokio-sync             0.1.7    0.1.8    Removed  Normal       ---
  tokio-signal                          0.2.7    0.2.9    0.2.9    Normal       ---
  tokio-signal->futures                 0.1.29   0.1.31   0.1.31   Normal       ---
  tokio-signal->libc                    0.2.96   0.2.97   0.2.97   Normal       cfg(unix)
  tokio-signal->mio                     0.6.21   0.6.23   0.6.23   Normal       ---
  tokio-signal->mio-uds                 0.6.7    0.6.8    0.6.8    Normal       cfg(unix)
  tokio-signal->signal-hook             0.1.13   Removed  Removed  Normal       cfg(unix)
  tokio-signal->tokio-executor          0.1.9    0.1.10   0.1.10   Normal       ---
  tokio-signal->tokio-io                0.1.12   0.1.13   0.1.13   Normal       ---
  tokio-signal->tokio-reactor           0.1.11   0.1.12   0.1.12   Normal       ---
  tokio-signal->winapi                  0.3.8    0.3.9    0.3.9    Normal       cfg(windows)
  tokio-sync->fnv                       1.0.6    1.0.7    1.0.7    Normal       ---
  tokio-sync->fnv                       1.0.6    1.0.7    Removed  Normal       ---
  tokio-sync->futures                   0.1.29   0.1.31   0.1.31   Normal       ---
  tokio-sync->futures                   0.1.29   0.1.31   Removed  Normal       ---
  tokio-tcp->bytes                      0.4.12   ---      Removed  Normal       ---
  tokio-tcp->futures                    0.1.29   0.1.31   Removed  Normal       ---
  tokio-tcp->iovec                      0.1.4    ---      Removed  Normal       ---
  tokio-tcp->mio                        0.6.21   0.6.23   Removed  Normal       ---
  tokio-tcp->tokio-io                   0.1.12   0.1.13   Removed  Normal       ---
  tokio-tcp->tokio-reactor              0.1.11   0.1.12   Removed  Normal       ---
  tokio-threadpool->crossbeam-deque     0.7.2    0.7.3    Removed  Normal       ---
  tokio-threadpool->crossbeam-queue     0.1.2    0.2.3    Removed  Normal       ---
  tokio-threadpool->crossbeam-utils     0.6.6    0.7.2    Removed  Normal       ---
  tokio-threadpool->futures             0.1.29   0.1.31   Removed  Normal       ---
  tokio-threadpool->lazy_static         1.4.0    ---      Removed  Normal       ---
  tokio-threadpool->log                 0.4.8    0.4.14   Removed  Normal       ---
  tokio-threadpool->num_cpus            1.12.0   1.13.0   Removed  Normal       ---
  tokio-threadpool->slab                0.4.2    0.4.3    Removed  Normal       ---
  tokio-threadpool->tokio-executor      0.1.9    0.1.10   Removed  Normal       ---
  tokio-timer->crossbeam-utils          0.6.6    0.7.2    Removed  Normal       ---
  tokio-timer->futures                  0.1.29   0.1.31   Removed  Normal       ---
  tokio-timer->slab                     0.4.2    0.4.3    Removed  Normal       ---
  tokio-timer->tokio-executor           0.1.9    0.1.10   Removed  Normal       ---
  tokio-udp->bytes                      0.4.12   ---      Removed  Normal       ---
  tokio-udp->futures                    0.1.29   0.1.31   Removed  Normal       ---
  tokio-udp->log                        0.4.8    0.4.14   Removed  Normal       ---
  tokio-udp->mio                        0.6.21   0.6.23   Removed  Normal       ---
  tokio-udp->tokio-codec                0.1.1    0.1.2    Removed  Normal       ---
  tokio-udp->tokio-io                   0.1.12   0.1.13   Removed  Normal       ---
  tokio-udp->tokio-reactor              0.1.11   0.1.12   Removed  Normal       ---
  tokio-uds->bytes                      0.4.12   ---      Removed  Normal       ---
  tokio-uds->futures                    0.1.29   0.1.31   Removed  Normal       ---
  tokio-uds->iovec                      0.1.4    ---      Removed  Normal       ---
  tokio-uds->libc                       0.2.96   0.2.97   Removed  Normal       ---
  tokio-uds->log                        0.4.8    0.4.14   Removed  Normal       ---
  tokio-uds->mio                        0.6.21   0.6.23   Removed  Normal       ---
  tokio-uds->mio-uds                    0.6.7    0.6.8    Removed  Normal       ---
  tokio-uds->tokio-codec                0.1.1    0.1.2    Removed  Normal       ---
  tokio-uds->tokio-io                   0.1.12   0.1.13   Removed  Normal       ---
  tokio-uds->tokio-reactor              0.1.11   0.1.12   Removed  Normal       ---
  toml->serde                           1.0.104  1.0.126  1.0.126  Normal       ---
  winapi->winapi-i686-pc-windows-gnu    0.4.0    ---      Removed  Normal       i686-pc-windows-gnu
  winapi->winapi-x86_64-pc-windows-gnu  0.4.0    ---      Removed  Normal       x86_64-pc-windows-gnu
  ws2_32-sys->winapi                    0.2.8    ---      Removed  Normal       ---
  ws2_32-sys->winapi-build              0.1.1    ---      Removed  Build        ---
  yaml-rust->linked-hash-map            0.5.2    0.5.4    0.5.4    Normal       ---
  ```
* 9 unsafe occurrences
* cargo cov report
  ```
  Filename                                                                            Regions    Missed Regions     Cover   Functions  Missed Functions  Executed       Lines      Missed Lines     Cover    Branches   Missed Branches     Cover
  -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  /rustc/5dc8789e300930751a78996da0fa906be5a344a2/library/std/src/thread/local.rs           5                 2    60.00%           3                 1    66.67%          28                 0   100.00%           0                 0         -
  src/body_parser/fixed_length.rs                                                          54                21    61.11%          11                 2    81.82%         110                32    70.91%           0                 0         -
  src/body_parser/text.rs                                                                  37                11    70.27%          11                 2    81.82%          75                12    84.00%           0                 0         -
  src/config/config.rs                                                                    114                56    50.88%          22                14    36.36%         161                73    54.66%           0                 0         -
  src/config/log.rs                                                                        13                 6    53.85%           1                 0   100.00%          16                 5    68.75%           0                 0         -
  src/config/user.rs                                                                       53                17    67.92%          13                 4    69.23%          83                13    84.34%           0                 0         -
  src/lib.rs                                                                                1                 0   100.00%           1                 0   100.00%           1                 0   100.00%           0                 0         -
  src/main.rs                                                                               2                 1    50.00%           2                 1    50.00%           4                 3    25.00%           0                 0         -
  src/message/response.rs                                                                  22                22     0.00%          16                16     0.00%         155               155     0.00%           0                 0         -
  src/message/serializer.rs                                                               105                20    80.95%          20                 1    95.00%         323                36    88.85%           0                 0         -
  src/message/stomp_message.rs                                                            174                38    78.16%          59                 7    88.14%         412                48    88.35%           0                 0         -
  src/parser.rs                                                                           283                65    77.03%          36                 6    83.33%         426                91    78.64%           0                 0         -
  src/persist/mlog.rs                                                                      26                10    61.54%           4                 0   100.00%          57                10    82.46%           0                 0         -
  src/session/ecg.rs                                                                       59                18    69.49%           7                 2    71.43%         106                21    80.19%           0                 0         -
  src/session/filter.rs                                                                    83                20    75.90%          13                 2    84.62%         130                24    81.54%           0                 0         -
  src/session/mq.rs                                                                        54                50     7.41%          11                 9    18.18%          86                70    18.60%           0                 0         -
  src/session/reader.rs                                                                   174               174     0.00%          14                14     0.00%         310               310     0.00%           0                 0         -
  src/session/stomp_session.rs                                                            225               198    12.00%          44                34    22.73%         353               274    22.38%           0                 0         -
  src/session/subscription.rs                                                              58                58     0.00%          10                10     0.00%         100               100     0.00%           0                 0         -
  src/system/limits.rs                                                                      1                 0   100.00%           1                 0   100.00%           9                 0   100.00%           0                 0         -
  src/util.rs                                                                              16                 1    93.75%           4                 1    75.00%          19                 3    84.21%           0                 0         -
  src/web_socket/ws_demunge.rs                                                            196                66    66.33%          28                 2    92.86%         550                56    89.82%           0                 0         -
  src/web_socket/ws_headers.rs                                                            115                41    64.35%          20                 3    85.00%         181                44    75.69%           0                 0         -
  src/web_socket/ws_response.rs                                                            35                 8    77.14%           6                 2    66.67%          65                11    83.08%           0                 0         -
  src/workflow/console/filter.rs                                                           11                10     9.09%           4                 3    25.00%          34                31     8.82%           0                 0         -
  src/workflow/console/ngin.rs                                                             91                40    56.04%          21                 8    61.90%         307               149    51.47%           0                 0         -
  src/workflow/destination/destination.rs                                                 214               139    35.05%          43                28    34.88%         451               230    49.00%           0                 0         -
  src/workflow/destination/server.rs                                                      117                82    29.91%          20                 7    65.00%         217               116    46.54%           0                 0         -
  src/workflow/http_router.rs                                                              35                33     5.71%           5                 4    20.00%          60                53    11.67%           0                 0         -
  src/workflow/sha_auth.rs                                                                 30                 4    86.67%           6                 0   100.00%          66                 3    95.45%           0                 0         -
  -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  TOTAL                                                                                  2403              1211    49.60%         456               183    59.87%        4895              1973    59.69%           0                 0         -
  ```
* cargo cov show in separated md file
  Not covered unsafes:
  * `src/workflow/http_router.rs:51`
  * `src/workflow/http_router.rs:68`
  * `src/workflow/http_router.rs:89`
  * Probably all `src/workflow/router.rs` (not available in coverage)
* 4 static muts that should be replaced with safe code
* cargo udeps
  ```
  unused dependencies:
  `romp v0.5.2 (/home/edoval/src/romp)`
  └─── dependencies
       ├─── "tokio-signal"
       └─── "totems"
  Note: These dependencies might be used by other targets.
        To find dependencies that are not used by any target, enable `--all-targets`.
  Note: They might be false-positive.
        For example, `cargo-udeps` cannot detect usage of crates that are only used in doc-tests.
        To ignore some dependencies, write `package.metadata.cargo-udeps.ignore` in Cargo.toml.
  ```
