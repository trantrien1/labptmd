# labptmd

Labtainer lab **mal-ai-model-analysis**: analyzing a tampered AI model (static plus dynamic, sandbox only).

Step-by-step student guide (Vietnamese): [HUONG_DAN_SINH_VIEN.md](HUONG_DAN_SINH_VIEN.md)

## Running it on the Labtainer VM

```bash
cd ~/labtainer/labtainer-student
imodule https://raw.githubusercontent.com/trantrien1/labptmd/main/mal-ai-model-analysis.tar
labtainer mal-ai-model-analysis
```

- On the first run Labtainer pulls the image `trantrien/mal-ai-model-analysis.analyst.student` from Docker Hub (about 4 GB), so it takes a while.
- Container login: user `ubuntu` / password `ubuntu`. The handout is in `docs/mal-ai-model-analysis.docx`.
- Fill in `~/result.txt`, check your answers with `checkwork`, and submit with `stoplab`.
- Restart the lab from scratch: `labtainer -r mal-ai-model-analysis`

## Troubleshooting

- `imodule: command not found`, or an error saying the lab needs a newer framework: run `update-labtainer.sh` and try again.
- The CPU needs AVX (TensorFlow 2.x). In VirtualBox/VMware, make sure AVX is passed through to the VM.
