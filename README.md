# CHORAS

This is the public repository for the The Community Hub for Open-source Room Acoustics Software. Follow the steps described in [setup_instructions.md](./setup_instructions.md) to run CHORAS using Docker!

<img width="1512" height="786" alt="Screenshot 2025-11-04 at 11 23 16" src="https://github.com/user-attachments/assets/b3eb28d6-8a19-49a8-a8a9-08f156b09ef9" />

## Note on simulation time
To make sure that your first simulations don't take forever, take note of the following settings that cause long simulation times:
- Diffusion Equation
  - **Material properties** with low absorption (start with _Upholstered concert chairs_ for prototyping)
  - A low value for **Characteristic length** (start with **3**, especially for a larger geometry (such as the Room2215 geometries))
  - A high value for **Impulse Response Length** (**Simulation length** is set to **IR Length**)
- DG
  - A high value for **Impulse Response Length** (start with 0.1 for prototyping)
  - A high value for **Frequency upper limit** (start with 100 for prototyping)
  - General tip: leave **Poly order**, **Points per wavelength**, and **CFL** untouched. :)

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
