<script>
    import { setContext, getContext, onDestroy, onMount } from 'svelte'
    import { getTransitionDurations, delay, isBrowser } from 'easyroute-core/lib/utils'

    export let transition = null
    export let forceRemount = false
    export let name = 'default'

    const SSR_CONTEXT = !isBrowser()
    const context = getContext('easyrouteContext')
    const currentDepth = context ? context.depth + 1 || 0 : 0
    const _router = context ? context.router : null
    const attrs = Object.assign({}, $$props)
    const passedClasses = attrs.class
    const transitionData = SSR_CONTEXT ?
      null :
      transition ?
        getTransitionDurations(transition) :
        null

    let currentComponent = null
    let routeData = {}
    let transitionClassName = ''
    let prevRouteId = null
    let previousRutePath = null
    let unsubscribe = undefined
    let outletElement = null

    if (!_router) {
        throw new Error('[Easyroute] RouterOutlet: no router instance found. Did you forget to wrap your ' +
            'root component with <EasyrouteProvider>?')
    }

    setContext('easyrouteContext', {
        depth: currentDepth,
        router: _router
    })

    async function changeComponent(component, currentRouteId) {
        if (prevRouteId === currentRouteId && !forceRemount) return
        if (!transitionData) {
            currentComponent = component
        } else {
            transitionClassName = `${transition}-leave-active ${transition}-leave-to`
            await delay(transitionData.leavingDuration + 10)
            transitionClassName += ` ${transition}-leave`
            await delay(5)
            transitionClassName = `${transition}-enter`
            transitionClassName = `${transition}-enter-active`
            currentComponent = component
            transitionClassName += ` ${transition}-enter-to`
            await delay(transitionData.enteringDuration + 10)
            transitionClassName = ''
        }
        prevRouteId = currentRouteId
    }

    async function pickRoute(routes) {
        const currentRoute = routes.find(route => route.nestingDepth === currentDepth)
        if (currentRoute) {
            let component
            if (name === 'default') component = currentRoute.component || currentRoute.components.default
            else component = currentRoute.components ? currentRoute.components[name] : null
            changeComponent(component, currentRoute.id)
            await delay(transitionData ? transitionData.leavingDuration : 0)
            routeData = _router.currentRoute
        } else {
            currentComponent = null
        }
    }

    function sanitizeAttrs() {
        delete attrs.to
        delete attrs.$$slots
        delete attrs.$$scope
        delete attrs.router
        delete attrs.transition
        delete attrs.forceRemount
        delete attrs.class
    }

    sanitizeAttrs()

    onDestroy(() => {
        unsubscribe && unsubscribe()
    })

    if (SSR_CONTEXT) {
        pickRoute(_router.currentMatched.getValue)
        routeData = _router.currentRoute
    }

    onMount(() => {
        if (!SSR_CONTEXT) unsubscribe = _router.currentMatched.subscribe((routes) => { pickRoute(routes) })
    })
</script>

<div
    bind:this={outletElement}
    class="easyroute-outlet{ passedClasses ? ` ${passedClasses}` : '' }{ transitionClassName ? ` ${transitionClassName}` : '' }"
    {...attrs}
>
    <svelte:component this={currentComponent} currentRoute={routeData} router={_router} outlet={outletElement} />
</div>
