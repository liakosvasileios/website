+++
date = '2025-06-30'
draft = false
title = 'RPC Calculator Vulnerability Writeup'
tags = ["vulnerability research", "rpc", "linux"]
+++

---

## Executive Summary

This report identifies and analyzes vulnerabilities found in a socket-based and RPC-powered calculator program implemented in C. The system includes three components:

- `client.c`
- `calc_client.c` (server-side RPC caller)
- `calc_server.c` (RPC service implementation)

While the project functions as intended for vector calculations, it lacks critical input validation, secure communication, and safe memory handling. The vulnerabilities are categorized by severity and accompanied by suggested mitigations.

---

## Vulnerability Summary Table

| ID   | Component      | Issue                                  | Severity | Risk Description                          |
|------|----------------|----------------------------------------|----------|-------------------------------------------|
| V1   | client.c       | Unchecked return values from `scanf`   | Medium   | May lead to undefined behavior            |
| V2   | client.c       | No lower bound check for vector length | Medium   | Could allow negative indexing or zero-size|
| V3   | client.c       | No validation of `recv()` output       | Medium   | Memory corruption or logic flaws          |
| V4   | client.c       | Unencrypted socket communication       | High     | Vulnerable to MITM attacks                |
| V5   | calc_server.c  | Static memory reuse in avg/prod        | Medium   | May cause data races or memory leaks      |
| V6   | calc_server.c  | No sanity limit on vector size         | Medium   | Integer overflow or DoS                   |
| V7   | calc_client.c  | Unvalidated `read()` calls             | High     | Incomplete or corrupted input data        |
| V8   | calc_client.c  | No bounds check on vector size         | Medium   | Same as V2, but in server                 |
| V9   | calc_client.c  | Static results not freed in `prod_r_1` | Low      | Memory management issue (minor leak)      |
| V10  | calc_client.c  | Missing signal handling                | Low      | Zombie processes due to unhandled SIGCHLD |

---

## Detailed Vulnerabilities and Mitigations

### client.c

**V1. Unchecked `scanf` Return Values**  
- **Description:** Failure to validate input using `scanf` may cause logic issues.  
- **Mitigation:**
  ```c
  if (scanf("%d", &choice) != 1) {
      fprintf(stderr, "Invalid Input\n");
      exit(1);
  }
  ```

**V2. Missing Lower Bound Check for Vector Size**  
- **Description:** `n <= 0` is not rejected, which causes undefined behavior.  
- **Mitigation:**
  ```c
  if (n <= 0 || n > MAX_SIZE) {
      fprintf(stderr, "Invalid vector length\n");
      exit(1);
  }
  ```

**V3. Blind Use of `recv()`**  
- **Description:** Does not verify number of bytes actually received.  
- **Mitigation:**
  ```c
  if (recv(sockfd, &res, sizeof(res), 0) != sizeof(res)) {
      fprintf(stderr, "Incomplete data received.\n");
      close(sockfd);
      return 1;
  }
  ```

**V4. Unencrypted Communication**  
- **Description:** Sends raw input/output data over the network.  
- **Mitigation:** Migrate to SSL/TLS using OpenSSL or similar library.

---

### calc_server.c

**V5. Static Memory Allocation With Potential Races**  
- **Description:** Uses static variables without thread safety.  
- **Mitigation:** Use dynamically allocated memory and manage freeing carefully.

**V6. No Upper Bound on `n`**  
- **Description:** Could result in very large allocations or integer overflows.  
- **Mitigation:**
  ```c
  if (argp->n <= 0 || argp->n > 1000) return NULL;
  ```

---

### calc_client.c

**V7. Unsafe Use of `read()`**  
- **Description:** Assumes `read()` will always return full data size.  
- **Mitigation:**
  ```c
  ssize_t read_exact(int fd, void *buf, size_t len) {
      size_t total = 0;
      while (total < len) {
          ssize_t n = read(fd, (char*)buf + total, len - total);
          if (n <= 0) return -1;
          total += n;
      }
      return total;
  }
  ```

**V8. No Bound Checking on `n`**  
- **Description:** As with `client.c`, `n` must be bounded to avoid DoS or logic errors.  
- **Mitigation:** Add the same check as in V6.

**V9. Memory Leak in `prod_r_1_svc()`**  
- **Description:** Memory not freed after sending data.  
- **Mitigation:** Manually free allocated memory or implement RPC-specific `*_free()` functions.

**V10. Zombie Child Processes**  
- **Description:** `waitpid()` is not comprehensive; children may remain zombies.  
- **Mitigation:**
  ```c
  signal(SIGCHLD, SIG_IGN);
  ```
  Or implement a proper signal handler.

---

## Testing Recommendations

| ID   | Test Case                                                                 |
|------|---------------------------------------------------------------------------|
| V1   | Input a letter (e.g., `a`) when prompted for integer                     |
| V2   | Input `n = -5` or `n = 0`                                                |
| V3   | Disconnect server mid-transfer; observe for crash or undefined behavior  |
| V4   | Use Wireshark to confirm plaintext traffic                               |
| V5   | Run concurrent requests; inspect memory/data overlap                     |
| V6   | Input `n = 999999999`; observe memory issues or crash                    |
| V7   | Send partial vector data manually (e.g., via `netcat`)                  |
| V8   | Repeat test for V2 on server side                                        |
| V9   | Run under Valgrind to check for memory leaks                             |
| V10  | Spawn multiple clients rapidly; use `ps` to detect zombie processes      |

---

## Recommendations Summary

- Always validate all external input, especially sizes and types.
- Avoid using static memory in networked/RPC systems; prefer dynamic allocation.
- Secure all communications with SSL/TLS or equivalent.
- Implement robust signal handling for process management.
- Use helper functions to ensure full buffer read/write (e.g., `read_exact()`).
- Enforce strict upper and lower bounds on all buffer sizes and allocations.

---