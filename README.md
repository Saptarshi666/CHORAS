# CHORAS

This is the public repository for the The Community Hub for Open-source Room Acoustics Software. Follow the steps described in [setup_instructions.md](./setup_instructions.md) to run CHORAS using Docker!

## Submodules

You'll find two submodules in this repository:

- frontend-v2
- backend

You **won't** need these if you simply want to run the Docker setup described in [setup_instructions.md](./setup_instructions.md). 

If you are interested in the underlying code anyway, run

```bash
git submodule update --init --recursive
```

## Example geometries

Examples of geometries compatible with CHORAS in the [example_geometries](./example_geometries/) folder.

## For room acoustics simulation back-end developers

If you are a developer of a room acoustics simulation back-end, please refer to the [backend readthedocs](https://choras-backend.readthedocs.io/en/latest/).

Note that if you have issues with cloning this repository (and its submodules), you can download the zipped repository via the releases page of this repository: <https://github.com/choras-org/CHORAS/releases>.