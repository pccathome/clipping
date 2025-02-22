<script setup>
import { onMounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import PageWrap from './components/PageWrap.vue'
import Header from './components/Header.vue'
import FooterInfo from './components/FooterInfo.vue'
import loadingIco from './components/LoadingIco.vue'

function init() {
    createObjects()
    controls()
    light()
    tick()
}

// Ref
const webgl = ref(null)

// Scene
const scene = new THREE.Scene()
scene.background = new THREE.Color('#000000')

// Resize
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}

const handleResize = () => {
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
}

// Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true, stencil: true })
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
renderer.localClippingEnabled = true

// Camera
const camera = new THREE.PerspectiveCamera(36, sizes.width / sizes.height, 0.1, 1000)
camera.position.set(4, 2, 7.5)
scene.add(camera)

// Controls
let control
function controls() {
    control = new OrbitControls(camera, renderer.domElement)
    control.enableDamping = true
    control.dampingFactor = 0.05
}

// Lights
function light() {
    scene.add(new THREE.AmbientLight(0xffffff, 3))

    const dirLight = new THREE.DirectionalLight(0xffffff, 6)
    dirLight.position.set(5, 10, 7.5)
    scene.add(dirLight)
}

let planes, planeObjects, object, BoxObject
planes = [
    new THREE.Plane(new THREE.Vector3(-1, 0, 0), 1),
    new THREE.Plane(new THREE.Vector3(0, -1, 0), 1),
    new THREE.Plane(new THREE.Vector3(0, 0, -1), 1),
    new THREE.Plane(new THREE.Vector3(1, 0, 0), 1),
    new THREE.Plane(new THREE.Vector3(0, 1, 0), 1),
    new THREE.Plane(new THREE.Vector3(0, 0, 1), 1)
]

function createPlaneStencilGroup(geometry, plane, renderOrder) {
    const group = new THREE.Group()
    const baseMat = new THREE.MeshBasicMaterial()
    baseMat.depthWrite = false
    baseMat.depthTest = false
    baseMat.colorWrite = false
    baseMat.stencilWrite = true
    baseMat.stencilFunc = THREE.AlwaysStencilFunc

    // back faces
    const mat0 = baseMat.clone()
    mat0.side = THREE.BackSide
    mat0.clippingPlanes = [plane]
    mat0.stencilFail = THREE.IncrementWrapStencilOp
    mat0.stencilZFail = THREE.IncrementWrapStencilOp
    mat0.stencilZPass = THREE.IncrementWrapStencilOp

    const mesh0 = new THREE.Mesh(geometry, mat0)
    mesh0.renderOrder = renderOrder
    group.add(mesh0)

    // front faces
    const mat1 = baseMat.clone()
    mat1.side = THREE.FrontSide
    mat1.clippingPlanes = [plane]
    mat1.stencilFail = THREE.DecrementWrapStencilOp
    mat1.stencilZFail = THREE.DecrementWrapStencilOp
    mat1.stencilZPass = THREE.DecrementWrapStencilOp

    const mesh1 = new THREE.Mesh(geometry, mat1)
    mesh1.renderOrder = renderOrder
    group.add(mesh1)

    return group
}

//createObjects
function createObjects() {
    // hdr
    const hdrEquirect = new RGBELoader().load('/photo_studio_01_1k.hdr', function (texture) {
        texture.mapping = THREE.EquirectangularReflectionMapping
        texture.encoding = THREE.sRGBEncoding
        texture.colorSpace = THREE.SRGBColorSpace
        texture.anisotropy = 64
        texture.magFilter = THREE.LinearFilter
        texture.minFilter = THREE.LinearMipMapLinearFilter
        texture.needsUpdate = true
    })

    // box
    const boxT = new THREE.BoxGeometry(1.99, 1.99, 1.99)
    const materialT = new THREE.MeshPhysicalMaterial({
        color: 0x080808,
        metalness: 1,
        roughness: 0,
        transmission: 0.6,
        specularIntensity: 1,
        specularColor: new THREE.Color(0x000000),
        envMap: hdrEquirect,
        envMapIntensity: 1,
        ior: 5,
        opacity: 0.4,
        side: THREE.DoubleSide,
        transparent: true
    })
    BoxObject = new THREE.Mesh(boxT, materialT)
    scene.add(BoxObject)

    // torus knot
    const geometry = new THREE.TorusKnotGeometry(1, 0.3, 220, 60)
    object = new THREE.Group()
    scene.add(object)

    const material = new THREE.MeshPhysicalMaterial({
        color: 0x000000,
        metalness: 0.2,
        roughness: 0.6,
        clippingPlanes: planes,
        shadowSide: THREE.DoubleSide
    })

    // add the color
    const clippedColorFront = new THREE.Mesh(geometry, material)
    clippedColorFront.castShadow = true
    clippedColorFront.renderOrder = 6
    object.add(clippedColorFront)

    planeObjects = []
    const planeGeom = new THREE.PlaneGeometry(2, 2)

    for (let i = 0; i < 6; i++) {
        const poGroup = new THREE.Group()
        const plane = planes[i]
        const stencilGroup = createPlaneStencilGroup(geometry, plane, i + 1)

        // plane is clipped by the other clipping planes
        const planeMat = new THREE.MeshStandardMaterial({
            color: 0x960600,
            metalness: 0.1,
            roughness: 0.75,
            clippingPlanes: planes.filter((p) => p !== plane),

            // clipIntersection: false,
            stencilWrite: true,
            stencilRef: 0,
            stencilFunc: THREE.NotEqualStencilFunc,
            stencilFail: THREE.ReplaceStencilOp,
            stencilZFail: THREE.ReplaceStencilOp,
            stencilZPass: THREE.ReplaceStencilOp
        })
        const po = new THREE.Mesh(planeGeom, planeMat)
        po.onAfterRender = function (renderer) {
            renderer.clearStencil()
        }

        po.renderOrder = i + 1

        object.add(stencilGroup)
        poGroup.add(po)
        planeObjects.push(po)
        scene.add(poGroup)
    }
}

// Clock for consistent animations
const clock = new THREE.Clock()
const tick = () => {
    const elapsedTime = clock.getElapsedTime()

    object.rotation.x = elapsedTime * 0.8
    object.rotation.y = elapsedTime * 0.4

    for (let i = 0; i < planeObjects.length; i++) {
        const plane = planes[i]
        const po = planeObjects[i]
        plane.coplanarPoint(po.position)
        po.lookAt(po.position.x - plane.normal.x, po.position.y - plane.normal.y, po.position.z - plane.normal.z)
    }

    renderer.render(scene, camera)

    control.update()

    window.requestAnimationFrame(tick)
}

onMounted(() => {
    webgl.value.appendChild(renderer.domElement)
    window.addEventListener('resize', handleResize)
    init()
})
</script>

<template>
    <PageWrap>
        <Header />
        <div class="outline-none w-full h-dvh" ref="webgl"></div>
        <FooterInfo>
            <template v-slot:first>&nbsp;</template>
            <template v-slot:second> Clipping </template>
            <template v-slot:github>
                <a href="https://github.com/pccathome/clipping" target="_blank" class="underline-offset-2">GitHub</a>
            </template></FooterInfo
        ></PageWrap
    >
</template>
