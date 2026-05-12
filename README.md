# grafana-dashboards

공식 문서 기반 Grafana 대시보드 모음.

## 구조

```
grafana-dashboards/
├── docs/          # 공식 문서에서 추출한 메트릭 정보
├── dashboards/    # 완성된 Grafana 대시보드 JSON
├── panels/        # 재사용 가능한 패널 템플릿
└── scripts/       # 유틸리티 스크립트
```

## 워크플로우

1. **문서화**: 공식 사이트 URL을 제공하면 `docs/`에 메트릭 정보 정리
2. **대시보드 작성**: 정리된 문서를 바탕으로 Grafana 대시보드 JSON 생성
3. **Import**: Grafana UI에서 `dashboards/*.json` 파일을 직접 import
