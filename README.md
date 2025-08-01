panic: runtime error: index out of range [0] with length 0

goroutine 1 [running]:
github.com/grafana/loki/v3/pkg/loki.validateSchemaRequirements(0xc000ca0000)
	/src/loki/pkg/loki/validation.go:33 +0x712
github.com/grafana/loki/v3/pkg/loki.(*Config).Validate(0xc000ca0000)
	/src/loki/pkg/loki/loki.go:348 +0x1e8c
main.main()
	/src/loki/cmd/loki/main.go:69 +0x625
