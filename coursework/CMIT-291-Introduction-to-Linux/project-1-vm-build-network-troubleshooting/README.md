# Project 1 - Building and Configuring a Linux Environment

**Course:** CMIT 291 Introduction to Linux
**Due:** Week 3 (individual)
**Status:** Complete - graded 100/100

## Assignment Summary

Build a Linux virtual machine and configure the base environment, demonstrating core Linux administration skills such as file permissions, package management, and basic system configuration.

## Status

Ubuntu Server was installed and configured as a VM in VMware Workstation on a Windows host, covering VM creation, basic Linux environment setup, and file/package administration tasks. Graded 100/100.

Mid-project, the VM lost all network connectivity. Troubleshooting traced the cause to VMware's NAT networking service (`VMware NAT Service` / `vmnetnat`) having stopped on the Windows host - not a misconfiguration inside the guest OS. Restarting the service on the host restored the VM's network connectivity, which was confirmed and documented in the graded submission.

## Folder Contents

| File / Folder | Purpose |
|---|---|
| `README.md` | This file |
| `assets/` | Screenshots - pending, to be supplied by student |

## Notes on Integrity

No UMGC assignment prompts or rubric text are reproduced here. This folder currently holds a summary of the completed, graded work rather than a full step-by-step methodology write-up - the detailed command-by-command narrative from the original submission has not yet been transcribed into this repo, so it's flagged here as a to-do rather than reconstructed from memory or guessed at.
