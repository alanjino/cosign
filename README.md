# cosign

Step 1: Create a new repository in github

Step 2: Clone the repository using "git clone"

Step 3: Create a sample go application ,for example "hello world".

Step 4: write a dockerfile for the application to build docker image.

Step 5: create a new directory ".github" for github workflow . In ".github" create workflow/docker-image.yaml

****************************************************************************************************************************
          
      docker-image.yaml


name: Docker Image CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:

  build:

    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write
      contents: read
      actions: read
      security-events: write
    env:
      REGISTRY: ghcr.io
      GH_URL: https://github.com
    steps:
      - name: Checkout GitHub Action
        uses: actions/checkout@v3
        # setup Docker build action
      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v2
      - name: Docker metadata
        id: metadata
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value={{sha}},enable=${{ github.ref_type != 'tag' }}
          flavor: |
            latest=true
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GIT_TOKENS }}
      - name: Build image and push to GitHub Container Registry
        uses: docker/build-push-action@v4
        with:
          tags: ${{ env.REGISTRY }}/${{ github.repository }}:${{ github.run_id }}
          labels: ${{ steps.metadata.outputs.labels }}
          push: true
      - name: Install cosign
        uses: sigstore/cosign-installer@main
      - name: Sign the images
        run: |
          cosign sign -y ${{ env.REGISTRY }}/${{ github.repository }}:${{ github.run_id }}
        env:
          COSIGN_EXPERIMENTAL: 1
      - name: Verify the pushed tags
        run: cosign verify ${{ env.REGISTRY }}/${{ github.repository }}:${{ github.run_id }} --certificate-identity ${{ env.GH_URL }}/${{ github.repository }}/.github/workflows/docker-image.yml@refs/heads/main  --certificate-oidc-issuer https://token.actions.githubusercontent.com
        env:
          COSIGN_EXPERIMENTAL: 1
      - name: Run Trivy in GitHub SBOM mode and submit results to Dependency Graph
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          format: 'github'
          output: 'dependency-results.sbom.json'
          image-ref: '.'
          github-pat: ${{ secrets.GIT_TOKENS }}

***************************************************************************************************************************


Step 6: Add your github token in github secrets. Use ${{ secrets.GIT_TOKENS }} to get the token where ever you want .

Step 7: Make "git commit" and "git push" to add the changes to the repository .
         


