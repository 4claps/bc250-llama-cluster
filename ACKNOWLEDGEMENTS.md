# Acknowledgements

This repository contains original Ansible automation and documentation, but it
depends on and learned from the following upstream projects. Their names and
licenses remain their own; this project's MIT license does not relicense them.

- [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) — inference
  runtime, Vulkan backend, and RPC backend; MIT licensed upstream.
- [rpf16rj/bc250-steamos-real-toolkit](https://github.com/rpf16rj/bc250-steamos-real-toolkit)
  — reference implementation and profile data used to understand BC250 CPU/GPU
  governor behavior. The toolkit is not installed or vendored here.
- [WinnieLV/bc250-cu-live-manager](https://github.com/WinnieLV/bc250-cu-live-manager)
  — separately downloaded, commit-pinned, checksum-verified runtime tool used
  for readback and explicitly gated WGP routing. Its source is not vendored.
- [bc250-collective/bc250_smu_oc](https://github.com/bc250-collective/bc250_smu_oc)
  — CPU SMU configuration tool; MIT licensed upstream and installed from a
  pinned upstream commit.
- [filippor/cyan-skillfish-governor](https://github.com/filippor/cyan-skillfish-governor)
  — BC250 GPU governor package; MIT licensed upstream and installed as a pinned
  Fedora package build.

The Real Toolkit and CU live-manager repositories did not advertise a license
through GitHub when this release was prepared. This repository therefore links
to or downloads those upstream works but does not copy or redistribute their
source. Numeric hardware profile values and command-line interfaces are
documented for interoperability and reproducibility.

Model names, model files, and model licenses belong to their respective
publishers. No GGUF model is distributed by this repository.
